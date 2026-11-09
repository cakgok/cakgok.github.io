+++
title = "What if an emergency spill slot needs an emergency spill slot?"
date = 2026-11-09
draft = false
description = "Making sure LLVM can access the emergency spill slot."
[taxonomies]
tags = ["llvm", "compilers", "codegen"]
+++

Storing a local variable on the stack is a relatively simple operation.

```
store value, [base + offset]
```
A stack store ordinarily uses a base register that holds a memory address, and the offset helps us find the required address.
For a nearby local variable, this is straightforward indeed:

```
store value, [sp + 64]
```

But what if the local variable is farther away?

```
store value, [sp + 1536]
```

The problem here is that, depending on the target architecture, this instruction may not be possible in a single step.

Consider the Xtensa ISA, which powers the very popular ESP32 microcontroller.\
Its 32-bit store instruction `S32I` has an 8-bit immediate field that is also scaled by 4.

What this means is that the instruction can directly address offsets of 0, 4, 8, ..., 1020 bytes relative to a base register. If the offset is outside that, the compiler must construct the address separately, by using another register.

```
movi  r_scratch, 1536
add   r_scratch, sp, r_scratch
s32i  r_val, r_scratch, 0
```

{% <aside> %} `MOVI`, on the other hand, encodes a signed 12-bit immediate field, so any value between -2048 and 2047 works fine. {% </aside> %}

Now we need a **scratch register** to calculate the address. The problem is that ordinary register allocation has already finished. Why not allocate this register earlier?

{% <aside> %} A scratch register is basically a register that you use for a temporary, throwaway calculation. {% </aside> %}

Because by the time we know we need it, it's already too late. The need for a scratch register comes up during **frame-index elimination**, where the compiler assigns real register+offset values to abstract stack slots (frame indices). The offsets are only known once the frame layout is finalized, and the layout can't be final before register allocation. So, the order is: register allocation, then finalize the frame layout, learn the offsets, and only then will we know which accesses need a scratch register.

At that point, LLVM can still create a new virtual register to hold the temporary value. However, since ordinary register allocation is already over, this virtual register cannot go through the normal register allocation. Instead, LLVM uses **register scavenging** to find a physical register for it.

Now, if you have a usable register at that point, business as usual and all is well. We can use that empty register as the scratch register.

## Under Register Pressure

With high register pressure, every general-purpose register may already hold a live value. To get the scratch register, the scavenger has to temporarily borrow an occupied register. To do this, we take whatever is in the register, store it in an **emergency spill slot**, and free up the register. Now, the register is usable for temporary calculation. When we are done with it, we take the saved value from the spill slot and put it back into the register.

```
s32i  r_tmp, sp, emergency_offset   ; 1. Save a live register to the emergency slot
movi  r_tmp, 1536                   ; 2. Use that register to calculate the address
add   r_tmp, sp, r_tmp
s32i  r_val, r_tmp, 0               ; 3. Perform the original store
l32i  r_tmp, sp, emergency_offset   ; 4. Restore the saved register
```

Now, this is all good, but what if your emergency spill slot is also out of reach of your immediate?

## The Emergency Slot Is Too Far

If we apply the same reasoning, you can see how we'll have a problem.

1. We have an offset that needs to be calculated.
2. We borrow a register and save its value in the emergency slot.
3. If the slot is also out of our immediate's reach, we need an emergency spill slot to calculate the offset.
4. ???

This circular logic leads to a crash in LLVM: **Incomplete scavenging after 2nd pass**.

The code makes it obvious in [`RegisterScavenging.cpp`](https://github.com/llvm/llvm-project/blob/4c22a9f755be90fa8ce385f589761033609fe66f/llvm/lib/CodeGen/RegisterScavenging.cpp#L473-L498) (simplified):
```cpp
bool Again = scavengeFrameVirtualRegsInBlock(MRI, RS, MBB);
if (Again) {
  // "Warning: Required two scavenging passes"
  Again = scavengeFrameVirtualRegsInBlock(MRI, RS, MBB);
  // The target required a 2nd run (because it created new vregs while
  // spilling). Refuse to do another pass to keep compiletime in check.
  if (Again)
    report_fatal_error("Incomplete scavenging after 2nd pass");
}
```
The comment basically says that this is a stopgap to keep compile times in check. But I think a third pass wouldn't have helped anyway, since each spill creates another out-of-range access.

This problem is not specific to Xtensa. If your target has small immediate fields, the problem becomes easier to hit.

## Why Is the Spill Slot Too Far?

Before we look at the failure modes, let's demystify how the emergency spill slot is addressed. We need a stable base register that can reach the spill slot. It can be the **stack pointer (SP)**, the **frame pointer (FP)** or a separate **base pointer (BP)**.

We first have to make some decisions, and some of these decisions are made by the target ABI, while others are made by `TargetFrameLowering` / `TargetRegisterInfo` hooks and carried out by the Prologue/Epilogue Inserter (PEI).

1. Where FP points: The ABI usually makes this decision. On x86, AArch64 and RISC-V, FP usually sits near the incoming SP, exceptions notwithstanding. On Xtensa, Mips and SystemZ, the FP is just a copy of SP right after the prologue, so it sits at the bottom. The decision LLVM makes here is whether there is an FP at all (`hasFP()`), plus whether a separate base pointer is needed.

2. Where the emergency slot goes: This one really is a layout choice. PEI asks the target, through a hook called `allocateScavengingFrameIndexesNearIncomingSP()`, whether to place scavenging slots first (near incoming SP) or last (near SP). The hook can of course be overridden.

{% <svg path="static/figures/spill-slot/frames.svg" position="center" text_valign="center" panels="FP at the top | FP == SP | With VLAs"> %} {% </svg> %}

3. How a slot is addressed: This is decided per frame index, in `getFrameIndexReference()` / `eliminateFrameIndex()`, and it's likely that this decision is often forced by the first two. When there are variable-length arrays (VLAs) in the stack frame, you can't use SP-relative access.\
\
On the other hand, if the stack frame needs realignment, FP can no longer be used to address the emergency slot. The reason is that the FP is copied from the SP *before* the realignment. Realignment moves SP down by an amount known only at run time, so the offset is not a compile-time constant.

4. Whether to reserve the call frame: Reserving it means the outgoing-argument area is allocated once in the prologue, sized for the largest call, instead of moving SP around each call. Not reserving means there's no gap, but the SP moves, and that movement needs to be accounted for.

So, we have four decisions that must agree. Each of these may look fine in isolation, but they still have to align with each other, else we get bugs. The failures might show up due to large stack frames, dynamic stack adjustments, or high register pressure.

Here are three different ways the spill slot can go out of our reach:

### A. It's on the Wrong End of the Stack Frame

{% <svg path="static/figures/spill-slot/modes-wrong-end.svg" position="left" text_valign="center"> %}

LLVM's default puts the slot near the *incoming SP* whenever there's an FP. This assumes the FP is up there too. If your FP is near the SP instead, you can see how this can lead to some issues.

If your frame is big enough, you just crashed the compiler.

With Xtensa, this is quite easy due to the limited range of the immediate, as we talked about.

{% </svg> %}

Here's a simple reproducer to crash the compiler:

```c
static int min(int a, int b) { return a < b ? a : b; }
void callee(int, int, int, int, int, int, int);
void use(char *);

void f(const int *p) {
  char big[2100];
  use(big);
  callee(min(p[0], p[1]), min(p[2], p[3]), min(p[4], p[5]), min(p[6], p[7]),
         min(p[8], p[9]), min(p[10], p[11]), 7);
}
```

```
$ clang --target=xtensa -O2 -S -emit-llvm repro.c -o - | llc -mtriple=xtensa -mcpu=esp32s3 -o -
LLVM ERROR: Incomplete scavenging after 2nd pass
```

{% <aside> %} Upstream only reserves the slot when the estimated frame size is at least 2048 bytes. Smaller frames get a different error, due to another bug: "Cannot scavenge register without an emergency spill slot". {% </aside> %}

But what exactly makes this code need a scratch register?\
First, this shows up with an FP, and clang happens to add one by default at this version.

Second, it's the call.\
The 7th argument doesn't fit in registers, so it goes on the stack, and Xtensa makes room for it by moving SP down right before the call.
Moving SP takes a temporary register:
```
addi  a8, a1, -4    ; a8 = SP - 4
movsp a1, a8        ; SP = a8
```
Xtensa has 16 general-purpose registers. The return address, SP and FP occupy 3 of them. Twelve values loaded from `p` and the constant 7 occupy the remaining 13. So the scavenger has to borrow one, and the emergency slot it saves it to sits more than 2 KiB above FP due to `big[2100]`.

{% <aside> %} Xtensa only reserves space for stack arguments in advance when there is no FP. We'll come back to this in section C. {% </aside> %}

The fix is simple: you just override the placement of the slot ([#237791](https://github.com/llvm/llvm-project/pull/237791)). This is what SystemZ and Mips do as well. [SystemZ's override](https://github.com/llvm/llvm-project/blob/4c22a9f755be90fa8ce385f589761033609fe66f/llvm/lib/Target/SystemZ/SystemZFrameLowering.h#L35-L39) comes with a comment that describes this exact case:

> The default implementation assumes an x86-like layout, where the frame pointer is at the opposite end of the frame from the stack pointer. This meant that when frame pointer elimination was disabled, the slots ended up being as close as possible to the incoming stack pointer, which is the opposite of what we want on SystemZ.

Here is the exact hook that does it:
```cpp
bool allocateScavengingFrameIndexesNearIncomingSP(
    const MachineFunction &MF) const override {
  return false;
}
```

---

There is another way the slot ends up on the wrong side, and that's with an over-aligned local. The gap between FP and the locals is unknown at compile time, so locals have to be addressed from SP (or another base pointer). A slot placed near FP would be addressed from SP, across the whole frame.

This case has come up before: when the stack is realigned, PEI places the slot near SP instead ([`cad78a763455`](https://github.com/llvm/llvm-project/commit/cad78a763455)). A base pointer can cause the same mismatch. In AArch64, functions with VLAs kept the slot at the top of the frame but addressed it from the base pointer at the other end, and the fix was to reach it from the FP ([`f2c22050e821`](https://github.com/llvm/llvm-project/commit/f2c22050e821)).

The default implementation of [`allocateScavengingFrameIndexesNearIncomingSP()`](https://github.com/llvm/llvm-project/blob/4c22a9f755be90fa8ce385f589761033609fe66f/llvm/lib/CodeGen/TargetFrameLoweringImpl.cpp#L172-L180) accounts for this. It only places the slot near the incoming SP when there is an FP and the stack is not realigned:

```cpp
bool TargetFrameLowering::allocateScavengingFrameIndexesNearIncomingSP(
  const MachineFunction &MF) const {
  if (!hasFP(MF))
    return false;

  const TargetRegisterInfo *RegInfo = MF.getSubtarget().getRegisterInfo();
  return RegInfo->useFPForScavengingIndex(MF) &&
         !RegInfo->hasStackRealignment(MF);
}
```

{% <aside> %} The hook is fairly new: it dates from 2021 and replaced an older knob. Funnily enough, the [commit](https://github.com/llvm/llvm-project/commit/273a0c8bc9c7) even says: "I don't understand what the existing control `isFPCloseToIncomingSP` is supposed to mean". {% </aside> %}

`useFPForScavengingIndex()` only guides the placement. Frame-index elimination still has to choose a suitable base.

---

### B. Offset Between the Base and the Slot

The slot is at the right end, but something is laid out between it and the base register. This usually happens when there is a large reserved outgoing call frame, because the slot is normally allocated right above the outgoing call area.

{% <svg path="static/figures/spill-slot/modes-call-frame.svg" position="center" text_valign="center" panels="Before @ 0-318 | After @ 318-720"> %} {% </svg> %}

With bottom placement, the slot comes after all locals, with the max call frame size below it.\
So, the slot sits exactly `MaxCallFrameSize` above SP.\
Above 2047 bytes on RISC-V ([#180199](https://github.com/llvm/llvm-project/issues/180199)) or 1020 on Xtensa, it's out of reach.

Two things are happening in the reproducer below. First, the 2100-byte struct goes into the reserved outgoing call area, which sits between SP and everything else, including the slot. Second, `flag = v12` stores to a local more than 2100 bytes above SP while all 14 values are live, so it needs a scratch register.

```c
struct big {
    char bytes[2100];
};
void callee(struct big);
void use(char *, char *);

int f(const int *in, int *out, const struct big *p) {
  char flag;
  char buf[300];
  int v0 = in[0]; int v1 = in[1]; int v2 = in[2];
  int v3 = in[3]; int v4 = in[4]; int v5 = in[5];
  int v6 = in[6]; int v7 = in[7]; int v8 = in[8];
  int v9 = in[9]; int v10 = in[10]; int v11 = in[11];
  int v12 = in[12]; int v13 = in[13];
  flag = v12;
  int r = v0 + v1 + v2 + v3 + v4 + v5 + v6 + v7 + v8 + v9 + v10 + v11 + v13;
  *out = r;
  use(&flag, buf);
  callee(*p);
  return r;
}
```

Here we omit the frame pointer to force Xtensa to reserve the outgoing argument area in the fixed frame. That puts the large call area between SP and the emergency slot.

```
$ clang --target=xtensa -O2 -fomit-frame-pointer -S repro.c
fatal error: error in backend: Incomplete scavenging after 2nd pass
```

RISC-V fixes this by forcing an FP ([#182124](https://github.com/llvm/llvm-project/pull/182124)). Since on RISC-V the FP points near the incoming SP (at the top of the frame), it won't be affected by the large reserved call frame.

{% <aside> %} The RISC-V fix says "This idea is borrowed from AArch64". [AArch64](https://github.com/llvm/llvm-project/commit/5c290dc20601) hit the same bug and fixed it in January 2018, eight years earlier. {% </aside> %}

Interestingly, an FP at the top doesn't make you immune either: on AArch64, SME hazard padding between the FP and the locals pushed the slot out of reach ([#142190](https://github.com/llvm/llvm-project/pull/142190)). This is quite specific, though; I don't know of other backends that insert padding there in a similar fashion.

Evidently, this approach won't work if you are also putting the frame pointer on the stack pointer, like Xtensa does, so we need something else.

---

In that case, another possible solution is to stop reserving the call frame above a certain size (e.g., 1020 for Xtensa), force an FP, and let the SP move around function calls. This would make the FP the anchor, and since there would be no reserved space, the slot would always be reachable.

A minor upside of this approach is a smaller fixed stack frame. But you do generate some extra instructions to move the SP, so it's not without a trade-off. An interesting consequence is that these stack adjustments may also need temporary registers, creating another kind of register pressure.

{% <aside> %} This is basically the reason behind the reproducer in section A. {% </aside> %}

---

### C. The Base Register Has Moved by the Time of the Spill

In this case, the static layout may be fine, but the stack pointer isn't where it was when the layout was finalized.

SP moves for two reasons:

{% <svg path="static/figures/spill-slot/modes-base-moved.svg" position="left" text_valign="center"> %}

First, VLAs move it by an amount known only at run time, so the backend cannot address the frame from SP at all.

Second, calls move it by a fixed amount: the call-frame size. Frame-index elimination tracks that amount so the compiler can correct SP-relative accesses inside the call sequence. This might make it seem like the SP is still usable as a base. And it actually is, except for the scavenger's own spill and reload.

{% </svg> %}

A slot that was close to SP in the static layout is now farther away by the call-frame size. Moreover, all outgoing arguments are live at that point, filling up the registers, which is exactly when the scavenger needs one.

I can't really show a C reproducer here for Xtensa, because in the current implementation, functions without a frame pointer reserve their call frame. Consequently, this particular problem (SP moving around calls while frame objects remain SP-relative) doesn't materialize.

However, ARM can still hit it, and they guard against it with an assertion. You can see it fire in [#98113](https://github.com/llvm/llvm-project/issues/98113). The issue has been open since 2024.

```c
struct big { int w[1100]; };
void callee(int, int, int, int, struct big, int, int, int, int, int, int, int, int, int, int, int, int);

void f(volatile int *q, const struct big *p) {
  int v0 = q[0], v1 = q[1], v2 = q[2], v3 = q[3], v4 = q[4], v5 = q[5], v6 = q[6], v7 = q[7], v8 = q[8], v9 = q[9];
  callee(v0, v1, v2, v3, *p, v3, v0, v7, v4, v1, v8, v5, v2, v9, v6, v3, v0);
  q[0] = v0; q[1] = v1; q[2] = v2; q[3] = v3; q[4] = v4; q[5] = v5; q[6] = v6; q[7] = v7; q[8] = v8; q[9] = v9;
}
```

```
$ clang --target=thumbv7m-none-eabi -O2 -S repro.c     # LLVM built with assertions
Assertion `TFI->hasReservedCallFrame(MF) && "Cannot use SP to access the emergency spill slot in " "functions without a reserved call frame"' failed.
```

The assertion sits right below a comment that explains it, in [`ARMBaseRegisterInfo.cpp`](https://github.com/llvm/llvm-project/blob/4c22a9f755be90fa8ce385f589761033609fe66f/llvm/lib/Target/ARM/ARMBaseRegisterInfo.cpp#L864-L867):

> `PEI::scavengeFrameVirtualRegs()` cannot accurately track `SPAdj` because the call frame setup/destroy instructions have already been eliminated.
 That means the stack pointer cannot be used to access the emergency spill slot when `!hasReservedCallFrame()`.

`SPAdj` is that call-frame offset we just talked about. Frame-index elimination uses it to correct SP-relative offsets, but [`RegisterScavenging.cpp`](https://github.com/llvm/llvm-project/blob/4c22a9f755be90fa8ce385f589761033609fe66f/llvm/lib/CodeGen/RegisterScavenging.cpp#L389) hard-codes `int SPAdj = 0`: the scavenger assumes SP hasn't moved.

Because it cannot account for the movement of the SP, the scavenger keeps using the old `[sp + offset]`. Instead of a crash, we can get a silent miscompilation. Whatever lives at that wrong address can get silently clobbered.

{% <aside> %} It's rare in practice. It needs an out-of-range offset with all registers live around a call. {% </aside> %}

In this particular ARM example, without assertions, the resulting code looks just fine. No outgoing argument gets clobbered; I think it's by pure luck in the instruction ordering. You can, however, hand-roll MIR that exposes the issue.

## Reserve the Call Frame or Not?

So without a reserved call frame, SP can't anchor the slot. Then why does ARM stop reserving it? Here's the reason, in [`ARMFrameLowering.cpp`](https://github.com/llvm/llvm-project/blob/4c22a9f755be90fa8ce385f589761033609fe66f/llvm/lib/Target/ARM/ARMFrameLowering.cpp#L363-L374):

```cpp
// It's not always a good idea to include the call frame as part of the
// stack frame. ARM (especially Thumb) has small immediate offset to
// address the stack frame. So a large call frame can cause poor codegen
// and may even makes it impossible to scavenge a register.
if (CFSize >= ((1 << 12) - 1) / 2)  // Half of imm12
  return false;
```

Wait a minute, that's part of the fix from section B? ARM stops reserving large call frames so the slot stays in reach, but it doesn't force an FP. Do you see why this is such a painful and recurring problem? We fixed one issue by not reserving the call frame, and now we're hitting the same problem because we *didn't* reserve it. Neither choice works on its own. We can't ask the stack pointer to do both jobs. It can either move around calls or anchor the slot.

The way out of this is to give the second job to a register that doesn't move. LLVM's default already does this: [`hasReservedCallFrame()`](https://github.com/llvm/llvm-project/blob/4c22a9f755be90fa8ce385f589761033609fe66f/llvm/include/llvm/CodeGen/TargetFrameLowering.h#L315-L317) returns `!hasFP(MF)`. SP only moves when there's also an FP. That's also why the second fix in B forces an FP. ARM overrides the default to fix B and loses that guarantee.

{% <aside> %} This was corrected in Thumb1, where it [forces a base pointer](https://github.com/llvm/llvm-project/blob/4c22a9f755be90fa8ce385f589761033609fe66f/llvm/lib/Target/ARM/ARMBaseRegisterInfo.cpp#L484-L489) whenever SP moves, "for correctness in cases where we need an emergency spill slot." {% </aside> %}

Why not always force a stable reference from an FP/BP then? Well, the cost is a register. If your target has many registers, losing one to the FP may not matter much. On Xtensa, with only 14 usable general-purpose registers, you probably don't want it in every function, considering you only need it where the call frame is too big for SP to reach across.

AArch64 and RISC-V already work this way. They force an FP only above 255 and 2047 bytes of call frame, respectively. On Xtensa, that would be functions with more than 1020 bytes of outgoing arguments.

---

## Takeaways

My understanding is that it all comes down to a few connected decisions.
- Place the emergency slot close to the base register used in addressing.
- That register must be a stable reference point.
- Reserving the outgoing call area, using an FP, or introducing a base pointer are ways to achieve this, depending on the target's layout.
- Unknowns at compile time force you to use an FP/BP, as the stack pointer becomes unreliable.

*I only briefly touched on other mechanisms like hazard padding, alignment and base pointers to keep the post in focus. There are still plenty of small edge cases left to explore.*

Next up, I'll look at how to decide whether to reserve an emergency spill slot at all.

---

Disclaimer: All reproducers were tested with LLVM main at [`4c22a9f755be`](https://github.com/llvm/llvm-project/commit/4c22a9f755be90fa8ce385f589761033609fe66f) (24.0.0git, built with assertions). Xtensa is an experimental target, builds need it explicitly enabled.
