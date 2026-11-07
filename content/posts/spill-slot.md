+++
title = "What if an emergency spill slot needs an emergency spill slot?"
date = 2026-11-02
draft = false
description = "Making the emergency spill slot mechanism usable."
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

Consider the Xtensa architecture that powers the very popular microcontroller ESP32.\
Its 32-bit store instruction `S32I` has an 8-bit immediate field that is also scaled by 4.

This means the instruction can directly address byte offsets of 0, 4, 8, …, 1020 relative to the base register. If the required offset exceeds this range, the compiler must construct the address separately using another register.

{% <aside> %} `MOVI`, on the other hand, encodes a signed 12-bit immediate field. So, a value between -2048 and 2047 works fine. {% </aside> %}

```
movi  r_scratch, 1536
add   r_scratch, sp, r_scratch
s32i  r_val, r_scratch, 0
```

{% <aside> %} A scratch register is basically a register that you use for a temporary, throwaway calculation. {% </aside> %}

Now we need a **scratch register** to calculate the address. The problem is that ordinary register allocation has already finished. Why not allocate this register earlier?

Well, actually you do get to anticipate the ordinary scratch register needs earlier. This scratch register need comes up during **frame-index elimination**, after the final offsets become available. Only after the frame layout is finalized do we have this information.

At that point, LLVM can still create a new virtual register to hold the temporary value. However, since ordinary register allocation is already over, this virtual register cannot go through the usual register allocation. Instead, LLVM uses **register scavenging** to find a physical register for it. The job is to find a temporary after allocation is over, for a job that allocation could not foresee.

Now, if you have a usable register at that point, business as usual and all is well. We can use that empty register as the scratch register.

## Under Register Pressure

With high register pressure, every general-purpose register may already hold a live value. To get the scratch register, the scavenger has to temporarily borrow an occupied register. To do this, we take whatever is in the register, store it in an **emergency spill slot**, and free up the register. Now, the register is usable for temporary calculation. When we are done with it, we take the saved value from the spill slot, and put it back into the register.

{% <aside> %} This temporary borrowing is distinct from an ordinary register-allocation spill. The register allocator spills a value because there are not enough registers to hold all live variables. {% </aside> %}

```
s32i  r_tmp, sp, emergency_offset   ; 1. Save a live register to the emergency slot
movi  r_tmp, 1536                   ; 2. Use that register to calculate the address
add   r_tmp, sp, r_tmp
s32i  r_val, r_tmp, 0               ; 3. Perform the original store
l32i  r_tmp, sp, emergency_offset   ; 4. Restore the saved register
```

Now this is all good, but what if your emergency spill slot is also out of reach of your immediate?

## The Emergency Slot Is Too Far

If we apply the same reasoning, you can see how we'll have a problem.

1. We had an offset that needs to be calculated.
2. We'll use the emergency spill slot to do that calculation.
3. If the slot is also out of our immediate's reach, we need an emergency spill slot to calculate the offset.
4. ???

This circular logic leads to a crash in LLVM: **Incomplete scavenging after 2nd pass**.

Code makes it obvious in [`RegisterScavenging.cpp`](https://github.com/llvm/llvm-project/blob/5f576704c2b701878c22c4841e68133cbfb5593e/llvm/lib/CodeGen/RegisterScavenging.cpp#L473-L498) (simplified):
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
This is basically a stopgap to keep compile times in check. But I think a third pass wouldn't have helped anyway, since each spill creates another out-of range access.

This problem is not specific to Xtensa, however. But if your target has a small bitfield for its immediate, the problem becomes easier to hit.

## Why Is the Spill Slot Too Far?

Before we look at the failure modes, let's demystify how the emergency spill slot is addressed.

We first have to make some decisions, and some of these decisions are made by the target ABI, while some are made by `TargetFrameLowering` / `TargetRegisterInfo` hooks and carried out by the Prologue/Epilogue Inserter (PEI).

1. Where FP points: The ABI usually makes this decision. On x86, AArch64 and RISC-V, FP usually sits near the incoming SP, exceptions notwithstanding. The decision LLVM makes here is whether there is an FP at all (`hasFP()`), plus whether a separate base pointer is needed.

2. Where the emergency slot goes: This one really is a layout choice. PEI asks the target, through a hook called `allocateScavengingFrameIndexesNearIncomingSP()`, whether to place scavenging slots first (near *incoming SP*) or last (near *SP*). The hook can of course be overridden.

3. How a slot is addressed: This is decided per frame index, in `getFrameIndexReference()` / `eliminateFrameIndex()`, and it's likely that this decision is forced. You need some stable reference point in order to be able to access the emergency spill slot. This can be the Stack Pointer (*SP*), the Frame Pointer (*FP*) if it exists, or a separate base pointer. When there are VLAs in the stack frame, you can't use SP-relative access.

4. Whether to reserve the call frame: Reserving it means SP stays put, but the call area sits between SP and the slot grows. Not reserving means there's no gap, but the SP moves, and that movement needs to be accounted for,

So, we have four decisions that must agree. Each of these decisions can look fine in isolation, while failing in their interaction. The failures might show up due to large stack frames, dynamic stack adjustments, or high register pressure.

So what makes this slot get allocated so far away from our reach? The invariant should be clear: At both the spill and reload, LLVM needs a way to access the emergency spill slot without requiring another unavailable scratch register.

Here are 3 different occasions:

### A. It's on the Wrong End of the Stack Frame

If, for some reason, the slot ends up near the *incoming SP*, but you put the *FP* near the *SP*, you can see how this can lead to some ~small~ issues.

{% <svg path="static/figures/spill-slot/modes-wrong-end.svg" position="left" text_valign="center"> %}

If your frame is big enough, you just crashed the compiler.

With Xtensa, this is quite easy due to the limited range of the immediate, as we talked about.

{% </svg> %}

Here's a simple reproducer to crash the compiler:

{% <aside> %} Upstream only reserves the slot when the estimated frame size is at least 2048 bytes. Smaller frames get a different error, due to another bug: "Cannot scavenge register without an emergency spill slot". {% </aside> %}

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

Tested with LLVM head at the time: [`5f576704c2b7`](https://github.com/llvm/llvm-project/commit/5f576704c2b701878c22c4841e68133cbfb5593e) (24.0.0git).

{% <aside> %} Xtensa only reserves space for stack arguments in advance when there is no FP. We'll come back to this in part C. {% </aside> %}

But what exactly makes this code need a scratch register? It's the call.\
The 7th argument doesn't fit in registers, so it goes on the stack, and Xtensa makes room for it by moving SP down right before the call.
Moving SP takes a temporary register:
```
addi  a8, a1, -4    ; a8 = SP - 4
movsp a1, a8        ; SP = a8
```
Xtensa has 16 general-purpose registers. The return address, SP and FP occupy 3 of them. Twelve values loaded from `p` and the constant 7 occupy the remaining 13. So the scavenger has to borrow one, and the emergency slot it saves it to sits more than 2 KiB above FP due to `big[2100]`.

The fix is simple: you just override the placement of the slot ([#237791](https://github.com/llvm/llvm-project/pull/237791)). This is what SystemZ and Mips do as well, because on both targets the FP is placed near the SP. [SystemZ's override](https://github.com/llvm/llvm-project/blob/5f576704c2b701878c22c4841e68133cbfb5593e/llvm/lib/Target/SystemZ/SystemZFrameLowering.h#L35-L39) comes with a comment that describes this exact case:

> The default implementation assumes an x86-like layout, where the frame pointer is at the opposite end of the frame from the stack pointer. This meant that when frame pointer elimination was disabled, the slots ended up being as close as possible to the incoming stack pointer, which is the opposite of what we want on SystemZ.

Here is the exact hook that does it:
```cpp
bool allocateScavengingFrameIndexesNearIncomingSP(
    const MachineFunction &MF) const override {
  return false;
}
```

---

There is another way that the slot ends up on the wrong side and that's with an over-aligned local. The gap between FP and the locals is unknown at compile time, so locals have to be addressed from SP (or another base pointer). A slot placed near FP would be addressed from SP, across the whole frame.

This case is all over LLVM over the years: when the stack is realigned, PEI places the slot near SP instead ([`cad78a763455`](https://github.com/llvm/llvm-project/commit/cad78a763455)). A base pointer can cause the same mismatch. In AArch64, functions with VLAs kept the slot at the top of the frame but addressed it from the base pointer at the other end, and the fix was to reach it from the FP ([`f2c22050e821`](https://github.com/llvm/llvm-project/commit/f2c22050e821)).

{% <aside> %} The hook is kind of new, dated 2021, replacing an older knob. Funnily the [commit](https://github.com/llvm/llvm-project/commit/273a0c8bc9c7) even says: "I don't understand what the existing control `isFPCloseToIncomingSP` is supposed to mean". {% </aside> %}

The default implementation of [`allocateScavengingFrameIndexesNearIncomingSP()`](https://github.com/llvm/llvm-project/blob/5f576704c2b701878c22c4841e68133cbfb5593e/llvm/lib/CodeGen/TargetFrameLoweringImpl.cpp#L173-L181) accounts for this. It only places the slot near the incoming SP when there is an FP and the stack is not realigned:

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

`useFPForScavengingIndex()` tells the placement. The target's frame-index elimination still has to choose a compatible base.

---

### B. Offset Between the Base and the Slot

The slot is at the right end, but something is laid out between it and the base register. This generally happens when there is a large reserved outgoing call frame, because generally the slot is reserved before the outgoing call area.

{% <svg path="static/figures/spill-slot/modes-call-frame.svg" position="center" text_valign="center"> %} {% </svg> %}

With bottom placement, the slot is after all locals with the max call frame size below it. \
So, the slot sits exactly `MaxCallFrameSize` above SP.\
Above 2047 bytes on RISC-V ([#180199](https://github.com/llvm/llvm-project/issues/180199)) or 1020 on Xtensa, it's out of reach.

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
  int r = 0 + v0 + v1 + v2 + v3 + v4 + v5 + v6 + v7 + v8 + v9 + v10 + v11 + v13;
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

Two things are happening here. First, the 2100-byte struct goes into the reserved outgoing call area, which sits between SP and everything else, including the slot. Second, `flag = v12` stores to a local more than 2100 bytes above SP while all 14 values are live, so it needs a scratch register.

RISC-V fixes this by forcing an FP ([#182124](https://github.com/llvm/llvm-project/pull/182124)). Since on RISC-V the FP points near the incoming SP (at the top of the frame), it won't be affected by the large reserved call frame.

{% <aside> %} Funnily, the RISC-V fix borrowed the approach from [AArch64](https://github.com/llvm/llvm-project/commit/5c290dc20601), which was faced with a similar issue. {% </aside> %}

Maybe there is a case where the FP is on top, and some padding pushes the offset to an unreachable value, but I haven't come across it.

Obviously, this approach won't work if you are also putting the frame pointer on the stack pointer, like Xtensa does.

---

In that case, a possible solution is to stop reserving the call frame above a certain size (e.g., 1020 for Xtensa) and also force an FP, and let the SP move around function calls. This would let you have the FP be the anchor, and since there would be no reserved space, the slot would always be reachable.

The upside of this approach is, of course, a smaller fixed stack frame. But you do generate some extra instructions to move the SP, so it's not without a trade-off. However, the real pain point is that these stack adjustments may also need temporary registers, creating the same kind of pressure we just tried to alleviate using a different layout.

---

### C. The Base Register Has Moved by the Time of the Spill

In this case, the static layout may be fine, but the stack pointer isn't where it was when the layout was finalized.

{% <svg path="static/figures/spill-slot/modes-base-moved.svg" position="left" text_valign="center"> %}

One way that happens is when SP is adjusted around calls. Without a reserved call frame, SP is lower by the call-frame size inside the call sequence.

A slot that was close to SP in the static layout is now that much farther away. Moreover, all outgoing arguments are live at that point, filling up the registers, which is exactly when the scavenger needs one.

{% </svg> %}

The results here are different whether we have an FP or not.\
If we don't have a frame pointer, we need another stable reference. This can be the Stack Pointer. However, if we don't account for the movement of the SP (SP decreases by C, so a slot previously at [sp + offset] must now be accessed at [sp + offset + C]), instead of a crash, we get a silent miscompilation.

I can't really show a C reproducer here for Xtensa, because in the current implementation, functions without a frame pointer reserve their call frame. Consequently, this particular problem (SP moving around calls while frame objects remain SP-relative) doesn't materialize.

However, ARM can still hit it.

ARM protects this with an assertion, and you can see that it fires here [#98113](https://github.com/llvm/llvm-project/issues/98113). The issue has been open since 2024.

```c
struct big { int w[1100]; };
void callee(int, int, int, int, struct big, int, int, int, int, int, int, int, int, int, int, int, int);

void f(volatile int *q, const struct big *p) {
  int v0 = q[0], v1 = q[1], v2 = q[2], v3 = q[3], v4 = q[4], v5 = q[5], v6 = q[6], v7 = q[7], v8 = q[8], v9 = q[9];
  callee(v0, v1, v2, v3, *p, v3, v0, v7, v4, v1, v8, v5, v2, v9, v6, v3, v0);
  q[0] = v0; q[1] = v1; q[2] = v2; q[3] = v3; q[4] = v4; q[5] = v5; q[6] = v6; q[7] = v7; q[8] = v8; q[9] = v9;
}
```

```bash
$ clang --target=thumbv7m-none-eabi -O2 -S r.c     # LLVM built with assertions
Assertion `TFI->hasReservedCallFrame(MF) && "Cannot use SP to access the emergency spill slot in functions without a reserved call frame"' failed.
```

Without assertions, the resulting code looks just fine. No outgoing argument gets clobbered; I think it's by pure luck in the instruction ordering. You can actually hand-roll a MIR that can expose the issue.

---

One fix is to force a frame pointer in situations where the stack pointer moves around calls, which some backends already do. But consider, if your target has many registers, losing one to the frame pointer might be perfectly fine. However, with a target like Xtensa, where you have only 14 usable general-purpose registers, it might not be the best idea.

---

Another fix would be keeping the call frame reserved because the scavenger doesn't account for SP adjustments.\
LLVM explains it in [`ARMBaseRegisterInfo.cpp`](https://github.com/llvm/llvm-project/blob/5f576704c2b701878c22c4841e68133cbfb5593e/llvm/lib/Target/ARM/ARMBaseRegisterInfo.cpp#L864-L867):

```cpp
// PEI::scavengeFrameVirtualRegs() cannot accurately track SPAdj because the
// call frame setup/destroy instructions have already been eliminated.  That
// means the stack pointer cannot be used to access the emergency spill slot
// when !hasReservedCallFrame().
```

And [`RegisterScavenging.cpp`](https://github.com/llvm/llvm-project/blob/5f576704c2b701878c22c4841e68133cbfb5593e/llvm/lib/CodeGen/RegisterScavenging.cpp#L389) hard-codes `int SPAdj = 0`.

Do you see why this is such a difficult and recurring problem? We fixed one issue by not reserving the call frame, and now you are telling me we are hitting the same problem because we *didn't* reserve the call frame?

---

The second way this happens is with variable-sized objects (alloca, VLAs). The dynamic area grows below the locals, so SP can end up any distance from the slot. Targets I checked were addressing the slot through FP or a base pointer here. A backend that addressed it from SP would be at the mercy of a run-time size.

## Takeaways

My understanding is that it all comes down to a few connected decisions.
- Reserve the emergency slot close to the base register used in addressing.
- That register must be stable, or its movement must be accounted for.
- Reserving the outgoing call area, using FP, or introducing a base pointer are ways to achieve this, depending on the target's layout.

*There are other mechanisms to consider like hazard padding or alignment, but I decided to focus only on core ideas that exist on every target*.

Next up, we'll look at how we can decide to actually reserve an emergency spill slot or not.
