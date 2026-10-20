+++
title = "Hunting a wrong-code bug in GCC's vectorizer"
date = 2026-10-18
draft = false
description = "Debugging and fixing a wrong-code bug in GCC's loop vectorizer (PR126225)."
[taxonomies]
tags = ["gcc", "compilers", "vectorization"]
+++

I encountered a bug during a coding competition of all places. You know the kind: you solve a algorithm problem, send your code to an online judge, and you get a verdict back. What do you do when you get back a "Wrong Answer" though? You blame ~~the compiler~~ yourself, of course. But what if Clang and GCC give you different outputs, which one do you trust?

In general, with a bug like this, most of the time it's undefined behavior. Not this time though.

It became [PR126225](https://gcc.gnu.org/bugzilla/show_bug.cgi?id=126225). I ended up writing the [fix](https://gcc.gnu.org/cgit/gcc/commit/?id=6c1a23f4bfe2403c12439c299064af557b317737) myself. It's been merged into trunk and backported to 16.2.

## Reproducer

The starting code already wasn't that complicated, so I stripped it down until I could get something minimal.

Here is the loop.
The `fill` function is also the first of the regression tests that went in with the fix.

```c
#include <assert.h>

__attribute__((noipa))
void fill(int *ans, int rounds)
{
  int x = 1;
  int y = 2;
  int i = 0;

  while (rounds > 0)
    {
      ans[i++] = x; ans[i++] = x; ans[i++] = x; ans[i++] = x;
      ans[i++] = y; ans[i++] = y; ans[i++] = y; ans[i++] = y;

      --rounds;
      x += 2;
      y += 2;
    }
}

int main(void)
{
    int ans[16] = {0};
    fill(ans, 2);

    const int expected[16] = {
        1, 1, 1, 1, 2, 2, 2, 2,
        3, 3, 3, 3, 4, 4, 4, 4
    };

    for (int i = 0; i < 16; ++i) assert(ans[i] == expected[i]);
}
```

With `rounds = 2` the answer should be:

```
expected  1 1 1 1 2 2 2 2 3 3 3 3 4 4 4 4
-O2       1 1 1 1 1 1 1 1 3 3 3 3 3 3 3 3
```

## Experimenting With Flags

{% <aside> %}_Vectorization_ means one instruction works on several values at once. A 128-bit SSE register (`xmm`) holds four `int32`s, a 256-bit AVX2 register (`ymm`) holds eight. Each slot is a _lane_.{% </aside> %}

I tried some more different shapes.
- With `int64`, it still produced wrong code.
- With `-mavx2` and `int32`s, it produced correct code, because AVX2 makes it use its wider 256-bit `ymm` registers.
- With `-mavx2` and `int64`, it's wrong again, because a 256-bit register only holds four 64-bit values, and the bug comes back.

| Flags                         | Element | Result  |
|-------------------------------|---------|---------|
| `-O1`                         | `int32` | correct |
| `-O2`                         | `int32` | wrong   |
| `-O2 -fno-tree-vectorize`     | `int32` | correct |
| `-O2 -mavx2`                  | `int32` | correct |
| `-O2`                         | `int64` | wrong   |
| `-O2 -mavx2`                  | `int64` | wrong   |
| `-O2 -fno-tree-slp-vectorize` | `int32` | wrong   |
| `-O2 -fno-tree-loop-vectorize`| `int32` | correct |

I also realized if you put a print statement inside the loop, the bug goes away. Moreover, at `-O1` it was correct. `-O2` was wrong, and `-O3` was correct again.

{% <aside> %}Two ways of vectorizing: the first is Superword-Level Parallelism (SLP), also called straight-line. It looks at side-by-side operations, and packs them into a vector operation if it can.{% </aside> %}

One thing that confused me was `-fno-tree-slp-vectorize`. Apparently, it only disables the separate basic-block SLP pass. The loop vectorizer's own SLP still runs, so it doesn't help.

{% <aside> %}The other is loop vectorization. It packs the same statement from different iterations into a single vector. In this example, the loop vectorizer uses SLP to pack operations within one iteration.{% </aside> %}

`-O3` threw me off too, but GCC has an attribute, `__attribute__((noipa))`, for exactly this purpose. It lets us disable interprocedural optimizations between functions, such as inlining. That's why `fill` has it in the reproducer above: with it, `-O3` is wrong too. Note that `__attribute__((noinline))` was not enough to bring the bug back at `-O3`.

The bug was present in all the major releases from 11.1 up to 16.1, which was the latest at that point. At `-O2` it only shows up from 12.1, because GCC 12 is when `-O2` started vectorizing.

`-fno-tree-loop-vectorize` already pointed me to something happening in the loop vectorizer.

## More Experiments

If you work with compilers a lot, at this point you probably know where to look. Source is there after all. But I didn't. Instead, I wrote a template program that tested the compiler output with different shapes. The template replaces the body with different patterns such as `xxxyyyyy`, signifying one letter per store. Every case is compiled at `-O1` as a reference and `-O2`, and the outputs are compared.

| Lane pattern | Flags | Result | -O2 output (wrong cases) |
| --- | --- | --- | --- |
| `xxxxyyyy` | SSE | wrong | `11111111 33333333` |
| `xxxyyyyy` | SSE | wrong | `11121112 33343334` |
| `xxxxxxxy` | SSE | wrong | `11111111 33333333` |
| `xyxyxyxy` | SSE | correct |  |
| `xxyyxxyy` | SSE | correct |  |
| `xxxxyyyy` | AVX2 | correct |  |

So, it's not really about the uniformity of the pattern.
The passing patterns (`xyxyxyxy`, `xxyyxxyy`) repeat every 4 elements, which ended up playing a part in understanding the bug. In hindsight, 4 is the number of `int32`s that fit in a 128-bit SSE register, which is why the register width and the element size mattered. I didn't see that yet, though.

## Looking at the Assembly

So, I checked the generated assembly. *(This is with 16.1, -O2)*

```asm
"fill":
    test    esi, esi
    jle     .L1
    lea     eax, [rsi-1]              # rounds - 1, for the end pointer
    pcmpeqd xmm0, xmm0                # xmm0 = {-1,-1,-1,-1}, all ones
    mov     edx, 2
    sal     rax, 5
    psrld   xmm0, 31                  # xmm0 = {1,1,1,1} <- x's start
    movd    xmm1, edx
    lea     rax, [rdi+32+rax]
    pshufd  xmm1, xmm1, 0             # xmm1 = {2,2,2,2} <- the STEP, not y
.L3:
    movups  XMMWORD PTR [rdi], xmm0   # ans[i+0..3] = xmm0
    add     rdi, 32
    movups  XMMWORD PTR [rdi-16], xmm0  # ans[i+4..7] = xmm0 <- should be y!
    paddd   xmm0, xmm1                # xmm0 += {2,2,2,2}
    cmp     rax, rdi
    jne     .L3
```

If you check it at `-O1`, you will see that it doesn't use `xmm` registers at all. It does scalar operations on general purpose registers.

{% <aside> %}There is also a future lesson here: I would have had an easier time if I had chosen more distinct values to investigate. It would have been so much easier to just dump everything and `grep` for that distinct value. {% </aside> %}

- Apparently, `pcmpeqd` + `psrld xmm0, 31` is a trick for building {1,1,1,1} without loading anything from memory. Compare a register with itself to get all-ones bits, then shift each 32-bit lane right by 31. Pretty cool.
- `mov edx, 2` + `movd` + `pshufd xmm1, xmm1, 0` fills `xmm1` with [2, 2, 2, 2].\
At first glance I thought this was y (which also starts at 2), but `xmm1` is only ever used to increment `xmm0` via `paddd`. So, it's the step instead.
- Looking at the stores in the loop body, we see that it writes `xmm0` to memory, shifts 16 bytes, and writes `xmm0` again before adding the step. (Well, actually, it shifts 32 bytes and writes to -16, but I digress.)

The point is, there should be a second vector but it's using `xmm0` for both halves of the array.

Suspicious.

## Checking the GIMPLE Dumps

{% <aside> %}GIMPLE is the intermediate representation (IR) of GCC where most of the optimizations happen. It's then passed off to the backend for code generation.{% </aside> %}

Since I already suspect that the bug is in the vectorizer, the plan is to check both sides: the GIMPLE given to the backend, and the GIMPLE going into the vectorizer.

First question to answer is this: Is the GIMPLE handed to the backend already wrong? If it is, this will clear the backend.

`-fdump-tree-optimized` is the flag we need to see what is passed.

{% <aside> %}In Static Single Assignment (SSA) form, every variable can only be assigned once. Each assignment gets a new numbered name: `x_58` and `x_50` are versions of `x`, and `_28` is a temporary.{% </aside> %}

{% <aside> %}A _PHI_ node is how the compiler handles a variable whose value depends on which path the code took to get here. In a loop, that's either entering it for the first time or jumping back from the previous iteration. The numbers in brackets are the blocks the values come from: `(3)` is before the loop, `(4)` is the loop itself.{% </aside> %}

```c
<bb 4>:
  # vect_vec_iv_.15_29 = PHI <_28(4), { 1, 1, 1, 1 }(3)>
  # ivtmp.23_53 = PHI <ivtmp.23_60(4), ivtmp.23_59(3)>
  _58 = (void *) ivtmp.23_53;
  MEM <vector(4) int> [(int *)_58]       = vect_vec_iv_.15_29;
  MEM <vector(4) int> [(int *)_58 + 16B] = vect_vec_iv_.15_29;
  _28 = vect_vec_iv_.15_29 + { 2, 2, 2, 2 };
  ivtmp.23_60 = ivtmp.23_53 + 32;
  if (_4 != ivtmp.23_60) goto <bb 4>;
```

- The PHI takes its initial value {1,1,1,1}, which is stored to bytes 0–15 and again to bytes 16–31, giving `1 1 1 1 1 1 1 1`.
- _28 = {3,3,3,3} and the pointer advances 32 bytes.
- Second iteration gives `3 3 3 3 3 3 3 3`. That's the wrong output from the reproducer.

As expected, it lines up with the assembly: the PHI is xmm0, the step is xmm1.

## GIMPLE Entering the Vectorizer

`-fno-tree-vectorize` eliminates the bug. However it doesn't prove the vectorizer is faulty. Maybe an earlier pass merged x and y and the vectorizer merely exposed it? So, we check if the IR that the vectorizer receives is correct. If it is, that narrows the search to the vectorizer or a later pass.

{% <aside> %} `-fdump-passes` dumps the list of optimization passes, and you can check the ordering. {% </aside> %}

The last dump before the vectorizer runs can be checked with: `-fdump-tree-ifcvt`.

Trimmed:
```c
<bb 3>:
  # rounds_57 = PHI <rounds_49(6), rounds_30(D)(5)>
  # x_58 = PHI <x_50(6), 1(5)>
  # y_59 = PHI <y_51(6), 2(5)>
  # i_60 = PHI <i_47(6), 0(5)>
  _3 = ans_33(D) + (i_60 * 4);    *_3  = x_58;
  _6 = ...      + 4 bytes;        *_6  = x_58;
  _9 = ...      + 8 bytes;        *_9  = x_58;
  _12 = ...     + 12 bytes;       *_12 = x_58;
  _15 = ...     + 16 bytes;       *_15 = y_59;
  _18 = ...     + 20 bytes;       *_18 = y_59;
  _21 = ...     + 24 bytes;       *_21 = y_59;
  _24 = ...     + 28 bytes;       *_24 = y_59;
  rounds_49 = rounds_57 + -1;
  x_50 = x_58 + 2;
  y_51 = y_59 + 2;
  i_47 = i_60 + 8;
```

{% <aside> %}An _induction variable_ (IV) is a variable that changes by a fixed step every iteration, like `x += 2`.{% </aside> %}

There are four PHI nodes (four scalar induction variables: `rounds`, `x`, `y`, `i`). `x_58` starts at 1 and `y_59` starts at 2. Eight consecutive stores form a contiguous 32-byte group. This proves that everything the vectorizer needs is here and correct.

## What Happens Inside the Vectorizer

We can check the whole vectorization pass with `-fdump-tree-vect-details`.

{% <aside> %}Scalar Evolution (SCEV) is how GCC summarizes a loop variable's behavior into a `{init, +, step}` progression, rather than chasing individual assignments around the loop body.{% </aside> %}

First, we look at the plan. This shows that the vectorizer's analysis is correct. Early in the pass (`vect_analyze_scalar_cycles`), the scalar evolution (SCEV) analysis recognizes each PHI as an induction. It separates x and y as two distinct IVs with different initial values.
```
step: -1,  init: rounds_30(D)   Detected induction.
step: 2,   init: 1              Detected induction.   <- x
step: 2,   init: 2              Detected induction.   <- y
step: 8,   init: 0              Detected induction.   <- i
```

The store group becomes a single SLP node (0x...4798) with 8 lanes. Its only child node is an induction-PHI node with 8 lanes (0x...4838), one per stored value:

```c
node 0x...4798 (max_nunits=4, refcnt=2) vector(4) int
op template: *_3 = x_58;
    stmt 0 *_3 = x_58;    stmt 4 *_15 = y_59;
    stmt 1 *_6 = x_58;    stmt 5 *_18 = y_59;
    stmt 2 *_9 = x_58;    stmt 6 *_21 = y_59;
    stmt 3 *_12 = x_58;   stmt 7 *_24 = y_59;
    children 0x...4838
node 0x...4838 (max_nunits=4, refcnt=2) vector(4) int
op template: x_58 = PHI <x_50(6), 1(5)>
    stmt 0..3 x_58 = PHI <x_50(6), 1(5)>
    stmt 4..7 y_59 = PHI <y_51(6), 2(5)>
    children (nil) (nil)

vectorization factor = 1
```

In SLP terms:
- `group_size = 8`. The node has 8 *lanes* [x x x x y y y y].
- `nunits = 4`. It targets 128-bit xmm registers that hold four 32-bit ints.
- `VF = 1`. Since a single scalar iteration already gives 8 values - enough to fill two full registers - it doesn't need unrolling.
- `nvects = 2`. Number of vectors. Each SLP node simply needs to emit 2 of them: lanes 0–3 and lanes 4–7.

This tree is correct. Lanes 4–7 are explicitly y_59 with init 2.\
Up until this point, the compiler hasn't lost track of y_59.

If we look at the transform:

```c
------>vectorizing SLP node starting from: x_58 = PHI <x_50(6), 1(8)>
transform induction phi.
------>vectorizing SLP node starting from: *_3 = x_58;
transform store.
add new stmt: MEM <vector(4) int> [(int *)vectp_ans.16_27] = vect_vec_iv_.15_29;
add new stmt: vectp_ans.16_77 = vectp_ans.16_27 + 16;
add new stmt: MEM <vector(4) int> [(int *)vectp_ans.16_77] = vect_vec_iv_.15_29;
```

This dump narrows the problem down.

{% <aside> %}A _vector IV_ is the same thing in vector form: `vect_vec_iv_` starts at {1,1,1,1} and gets {2,2,2,2} added each iteration.{% </aside> %}

Looking at what `transform store` and `transform induction phi` just emitted, we can see that it generates two 16-byte vector stores (ans[0..3] at vectp_ans.16_27, and ans[4..7] at + 16), which is expected. However, both stores write the exact same `vect_vec_iv_.15_29` variable.

There is no IV starting at { 2, 2, 2, 2 } anywhere. The only { 2, 2, 2, 2 } is the step: the IV for y was never built.

## Bisecting

If you are familiar with GCC internals, you probably know that only two functions print `transform induction phi.`: `vectorizable_induction` and `vectorizable_nonlinear_induction`. There is nothing _nonlinear_ about the reproducer.

If you are like me and you don't know where this happens, you go ahead and bisect. With 10~15 steps, I was able to bisect it to this commit: [092cdbd](https://gcc.gnu.org/cgit/gcc/commit/?id=092cdbd9198497)

Perfect.

The commit only touches `vectorizable_induction`, in `tree-vect-loop.c` (renamed to `tree-vect-loop.cc` in 2022).

{% <aside> %}CSE: Common subexpression elimination. Reusing a value instead of computing it twice, which is exactly what went wrong.{% </aside> %}

The commit is called "Re-instantiate SLP induction IV CSE". It puts back in a shortcut that was previously removed. It's from November 2020, so it first shipped in GCC 11.1, which matches the versions above.

The analysis and transform halves of `vectorizable_induction` are separated in the function. This also matches with our previous findings too, because the analysis was already correct, and the commit only changed the transform.

## Reading the Source

To vectorize the loop, `vectorizable_induction` builds a vector IV for
each chunk (one vector's worth of lanes). Here that means two vector IVs, one for the `x` lanes and one for the `y` lanes. Each vector IV costs a register and an add per iteration, so the vectorizer tries not to build more than it needs. Before the fix, the count was computed like this (in `gcc/tree-vect-loop.cc`):

{% <aside> %}`nivs` is basically the "Number of Induction Vectors". `ivn` is just the loop counter.{% </aside> %}

```c
/* Compute the number of distinct IVs we need.  First reduce
   group_size if it is a multiple of const_nunits so we get
   one IV for a group_size of 4 but const_nunits 2.  */
unsigned group_sizep = group_size;
if (group_sizep % const_nunits == 0)
  group_sizep = group_sizep / const_nunits;
nivs = least_common_multiple (group_sizep, const_nunits) / const_nunits;
```
As the comment says, this is the number of distinct IVs to build.
For our loop, `group_size` is 8 and `const_nunits` (lanes per vector) is 4.\
`group_sizep` becomes 2.\
`least_common_multiple (2, 4) / 4` is **1**.

GCC decided one vector IV was enough. Later, when it filled in the vector definitions for the remaining chunks, it copied the first IV into every slot:

{% <aside> %}“Wait, it’s all modular arithmetic?”<br>“Always has been.”{% </aside> %}

```c
/* Fill up to the number of vectors we need for the whole group.  */
nivs = least_common_multiple (group_size, const_nunits) / const_nunits;
for (; ivn < nivs; ++ivn)
  {
    slp_node->push_vec_def (SLP_TREE_VEC_DEFS (slp_node)[0]);
    vec_steps.quick_push (vec_steps[0]);
  }
```

Here `nivs` is recalculated from the full `group_size`, not `group_sizep`, so it's 2. `ivn` is 1 after building the one IV, so the loop runs once and fills the second slot.

So the second vector, with the `y` lanes, got the IV that was built for the first vector, with the `x` lanes.

However, this reduction is only correct when every chunk is a copy of an earlier one. The same initial values and the same steps, lane for lane. The earlier comment (a group of 4 with 2 lanes per vector) works when the second pair of lanes actually repeats but nothing was checking that it did.

But wait, there is a second problem in the fill: it always copies IV 0 (`vec_steps[0]`). Which is ok when only one IV is built. When there are more though, chunks that should reuse the second or third IV get the first one instead. So even a valid reduction can give wrong code, whenever a later chunk needs an IV other than the first. So this becomes another piece to patch.

## The Fix

We have to change two things.

First, compute the unreduced count - which is always correct - and treat the reduced count only as a _candidate_:

```c
nivs = least_common_multiple (group_size, const_nunits) / const_nunits;
unsigned group_sizep = group_size;
if (group_sizep % const_nunits == 0)
  {
    group_sizep = group_sizep / const_nunits;
    unsigned candidate_nivs
      = least_common_multiple (group_sizep, const_nunits) / const_nunits;
    if (candidate_nivs < nivs
        && vect_slp_induction_reuse_p (steps, inits, group_size,
                                       const_nunits, candidate_nivs, nivs))
      nivs = candidate_nivs;
  }
```

We create a helper: `vect_slp_induction_reuse_p` that checks the shortcut is valid. \
For every chunk past the candidate count, it compares each lane's initial value and step against the lane at the same position in the chunk it would reuse
(`ivn % candidate_nivs`):

```c
for (unsigned ivn = candidate_nivs; ivn < nivs; ++ivn)
  {
    unsigned reuse_ivn = ivn % candidate_nivs;
    for (unsigned HOST_WIDE_INT eltn = 0; eltn < const_nunits; ++eltn)
      {
        unsigned HOST_WIDE_INT elt = ivn * const_nunits + eltn;
        unsigned HOST_WIDE_INT reused_elt = reuse_ivn * const_nunits + eltn;

        if (!operand_equal_p (steps[elt], steps[reused_elt], 0)
            || !operand_equal_p (inits[elt], inits[reused_elt], 0))
          return false;
      }
  }
return true;
```

If any lane differs, we throw away the candidate, keep the full count and build a separate IV for each chunk.

Secondly, the loop that fills in the remaining chunks now reuses IVs
_cyclically_ rather than always taking the first one:

```c
/* Fill up to the number of vectors we need for the whole group.  */
nivs = least_common_multiple (group_size, const_nunits) / const_nunits;
unsigned generated_nivs = ivn;   /* IVs actually built */
for (; ivn < nivs; ++ivn)
  {
    unsigned reuse_ivn = ivn % generated_nivs;
    slp_node->push_vec_def (SLP_TREE_VEC_DEFS (slp_node)[reuse_ivn]);
    vec_steps.quick_push (vec_steps[reuse_ivn]);
  }
```

This matters once the reduced count is more than one. We have to fill it cyclically. Take a group of 24 lanes laid out as `X Y X X Y X`, four lanes per chunk. The pattern repeats every three chunks, so three IVs cover all six chunks, but chunk 4 has to reuse IV 1 (the `Y` one), not IV 0.

With a single IV, `ivn % 1` is always 0, so the old loop and the new one do the same thing.

## Regression Tests

Three new tests went into the patch.

- `vect-iv-12.c`: chunks with **different inits** (the reproducer).
- `vect-iv-13.c`: chunks with the same inits but **different steps**

In both, the IV reuse must be rejected.

- `vect-iv-14.c`: the `X Y X X Y X` checks the cyclic reuse pattern when reuse is _valid_.

We check if it takes the shortcut, through a new dump message (`reusing 3 SLP induction IVs for 6 vector chunks`), and that the cyclic reuse gives the right values.

## Missed Optimization and Beyond

Is there missed optimization here? Maybe. Generally speaking, we could do better with `nivs = 2` since we only have X and Y shapes. However, the code can only reuse chunk `ivn % candidate_nivs`, so it has no way of expressing that.

Even then, the code at that point is in such a weird shape: an SLP group of IVs at least two vectors long, where the chunks repeat every three but only come in two distinct shapes. Even reading that is a handful. Probably not worth dwelling too much on.

---

Once I sent the patch, the turnaround was extremely quick. After a small back and forth, it has been merged, and is currently being backported to all supported releases. All in all, a fun debugging experience. Interesting to see how this bug survived such a long time, seeing that the eventual reproducer is extremely simple.

_Bootstrapped and regtested on x86_64-pc-linux-gnu._
