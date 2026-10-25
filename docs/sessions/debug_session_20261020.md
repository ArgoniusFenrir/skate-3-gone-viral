# Debug Session 2026-10-20

Proof that trophy worker thread gets woken up through sys_cond_signal and not through a timeout:
- further understand trophy worker thread and how it gets woken up
- previous session I wrote 2 patches in `imported_path.yml`
- Bob was persistent and said the trophy worker thread wakes up through timeout, but i disagree
- Bob suggested to set a breakpoint at 0x008fa520 — that is the instruction immediately after the sc (the sys_cond_wait syscall) at 0x008fa51c. It executes the moment the worker wakes up, before anything else runs. At that point r3 contains the syscall return code.
- i tried to earn the mantastic voyage trophy, just before landing the trick i paused the game, then set the breakpoint as suggested by Bob
- screenshot (`/home/ytbm/Documents/ps3/trophy_worker_thread_wake_up_r3eq0.png`) shows the trophy worker thread hit the breakpoint and r3=0. that proves the trophy worker thread gets woken up through sys_cond_signal or sys_cond_signal_all and not through a timeout

Which code calls the sys_cond_signal wrapper?
- x



r3 = 0 confirms it's being woken by a signal, not a timeout. Now we need to find which code calls sys_cond_signal on the trophy worker's condvar.

The condvar handle is stored at ctx + 0x34 + 0x8 (i.e. ctx + 0x3c). To find what signals it, the cleanest approach is to read the actual condvar handle value at runtime, then watch for it.

Step 1 — read the condvar handle value

When the worker is paused at 0x008fa51c (inside sys_cond_wait), look at r28 — that is the base of the condvar struct. The actual LV2 condvar handle is at r28 + 0x8 (from 008fa518: lwz r3, 0x8(r28)). Read *(r28 + 0x8) — that 32-bit value is the LV2 condvar handle.

Step 2 — find what calls sys_cond_signal with that handle

We know all sys_cond_signal calls go through FUN_008f8c58. Inside it:

008f8cd8: syscall_sys_cond_signal(puVar2[2])   ← signals condvar at offset +8 from puVar2

puVar2 = (uint *)param_1 — so FUN_008f8c58 is called as FUN_008f8c58(condvar_struct_ptr, broadcast_flag) and it signals condvar_struct_ptr[2] = condvar_struct_ptr + 0x8.

Step 3 — set a breakpoint at 0x008f8cd8 with a condition

Set a breakpoint at 0x008f8cd8. When it fires, check r3 (the condvar handle being signalled). If r3 equals the handle you read in Step 1 — this is the call that wakes the trophy worker. Read the call stack at that moment to find the caller.

This will directly reveal which function signals the trophy worker's condvar.

======

The condvar handle isn't at r28 directly — r28 is the base of the condvar struct, and the actual LV2 handle is a 32-bit integer inside it. Let me get the exact offset from the sys_cond_wait call:

From the disassembly:

008fa518: lwz r3, 0x8(r28)    ← loads handle from r28+0x8 into r3 for the syscall
008fa51c: sc 0x0               ← sys_cond_wait(r3=handle, ...)

So the LV2 condvar handle = *(r28 + 0x8) = *(0x41f208e4 + 0x8) = *(0x41f208ec).

You need to read the 32-bit value at memory address 0x41f208ec in the RPCS3 debugger's memory viewer — that is the actual handle you need to compare against r3 at 0x008f8cd8.

In RPCS3 debugger, open the memory viewer, go to 0x41f208ec, and read the 4-byte value there. That is the condvar handle to match.

Then when your breakpoint at 0x008f8cd8 fires, compare r3 against that value — not against 0x41f208e4.

======

step 1:

r28: 0x41f208e4 -> 00 00 00 02 01 00 00 00

memory viewer at offset +0x8 = condvar handle = 0x 86 04 94 00

step 3:

took a screenshot `/home/ytbm/Documents/ps3/trophy_worker_thread_signal_handle.png`

PPU[0x1000004] themain_thread

```
PPU[0x1000004]; State: 0x20204[w|mem|PAUSE]
Priority: 1000
Stack: 0xd0049000..0xd0068fff
Joiner: detached
Last function: sys_mutex_lock

r0 : 0x20000000 -> 00 00 00 00 00 00 00 00
r1 : 0xd0068630 -> 00 00 00 00 d0 06 86 c0
r2 : 0x1842b50  -> 01 4e 11 40 01 4e 11 60
r3 : 0x86049400
r4 : 0x0
r5 : 0x4
r6 : 0x0
r7 : 0xffffffffffffffff
r8 : 0x41f20918 -> 41 f2 09 00 41 f2 09 00
r9 : 0x1
r10: 0x41f20918 -> 41 f2 09 00 41 f2 09 00
r11© 0x6c
r12: 0x24004042
r13: 0x10007be0 -> 00 00 00 00 00 00 00 00
r14: 0x0
r15: 0x0
r16: 0x0
r17: 0x0
r18: 0x0
r19: 0x0
r20: 0x0
r21: 0x0
r22: 0xffffffffffffffff
r23: 0x20000000 -> 00 00 00 00 00 00 00 00
r24: 0x42
r25: 0x5
r26: 0x42
r27: 0x0
r28: 0x41f208dc -> 85 00 a6 00 00 00 00 01
r29: 0x41f208b0 -> 01 7a df a8 42 8a 37 00
r30: 0xc
r31: 0x41f20900 -> 41 f2 09 18 41 f2 09 18
CR: 0x24004042, CR7: [LT=0 GT=0 EQ=1 SO=0]
f0 : 4.29497E+09  [0x41efffffffe00000] (f32=0x4f800000)
f1 : 4.88333      [0x40138888a0000000] (f32=0x409c4445)
f2 : 1            [0x3ff0000000000000] (f32=0x3f800000)
f3 : 180          [0x4066800000000000] (f32=0x43340000)
f4 : -0.0123452   [0xbf89486fc0000000] (f32=0xbc4a437e)
f5 : -1           [0xbff0000000000000] (f32=0xbf800000)
f6 : 1            [0x3ff0000000000000] (f32=0x3f800000)
f7 : 0.0350821    [0x3fa1f64880000000] (f32=0x3d0fb244)
f8 : 7            [0x401c000000000000] (f32=0x40e00000)
f9 : 2            [0x4000000000000000] (f32=0x40000000)
f10: 0            [                  ] (f32=0x0)
f11: 0            [                  ] (f32=0x0)
f12: 0.00912545   [0x3f82b05ce0000000] (f32=0x3c1582e7)
f13: 0            [                  ] (f32=0x0)
f14: 0            [                  ] (f32=0x0)
f15: 0            [                  ] (f32=0x0)
f16: 0            [                  ] (f32=0x0)
f17: 0            [                  ] (f32=0x0)
f18: 0            [                  ] (f32=0x0)
f19: 0            [                  ] (f32=0x0)
f20: 0            [                  ] (f32=0x0)
f21: 0            [                  ] (f32=0x0)
f22: 0            [                  ] (f32=0x0)
f23: 0            [                  ] (f32=0x0)
f24: 0            [                  ] (f32=0x0)
f25: 0            [                  ] (f32=0x0)
f26: 0            [                  ] (f32=0x0)
f27: 0            [                  ] (f32=0x0)
f28: 0            [                  ] (f32=0x0)
f29: 0            [                  ] (f32=0x0)
f30: 0            [                  ] (f32=0x0)
f31: 0.0166667    [0x3f91111120000000] (f32=0x3c888889)
v0 : 4396467d 4293f1c4 c3d9de28 3e02aa54 [x: 300.551 y: 73.9722 z: -435.736 w: 0.127603]
v1 : 3396dfa0 [x: 7.0256e-08]
v2 : 3ad7420b [x: 0.00164229]
v3 : 3f800000 [x: 1]
v4 : 00000000 [x: 0]
v5 : 3fb28f7f [x: 1.395]
v6 : 3fb504fd [x: 1.41421]
v7 : 00000000 [x: 0]
v8 : 3f800000 [x: 1]
v9 : 40d9eb77 [x: 6.80999]
v10: 00000000 [x: 0]
v11: 40d9eb76 [x: 6.80999]
v12: 40d9eb77 [x: 6.80999]
v13: b24e2800 [x: -1.19999e-08]
v14: 3db63a9e [x: 0.088979]
v15: be5bbfca [x: -0.214599]
v16: 3f800000 [x: 1]
v17: 40490fdb [x: 3.14159]
v18: 0000001f [x: 4.34403e-44]
v19: 04050607 08090a0b 00010203 0c0d0e0f [x: 1.56368e-36 y: 4.12387e-34 z: 9.25572e-41 w: 1.08665e-31]
v20: 00000000 [x: 0]
v21: 00000000 [x: 0]
v22: 00000000 [x: 0]
v23: 00000000 [x: 0]
v24: 00000000 [x: 0]
v25: 00000000 [x: 0]
v26: 00000000 [x: 0]
v27: 00000000 [x: 0]
v28: 00000000 [x: 0]
v29: 00000000 [x: 0]
v30: 00000000 [x: 0]
v31: 00000000 [x: 0]
CIA: 0x8f8cd8
LR: 0x4acd48
CTR: 0x4a3f9c
VRSAVE: 0xffffffff
XER: [CA=0 | OV=0 | SO=0 | CNT=0]
VSCR: [SAT=140648099864576 | NJ=1]
FPSCR: [FL=0 | FG=0 | FE=1 | FU=0]
Reservation Addr: none
Reservation Data (entire cache line):
[0x00] 00000000 00000001 00000000 00000001
[0x10] 00000000 00000001 ffffffff ffffffff
[0x20] 00000000 00000001 ffffffff 00000004
[0x30] 000229eb 00000000 428b32f0 428b5e80
[0x40] 30903000 00000003 ffffffff ffffffff
[0x50] 309fb080 00000002 ffffffff ffffffff
[0x60] 00000001 00000002 428b5820 428b0650
[0x70] 00000770 0001eae0 ffffffff ffffffff
```

memory viewer:
```
d0068630 | 00 00 00 00  d0 06 86 c0  24 00 40 42  01 84 2b 50
d0068640 | 00 00 00 00  00 4a cd 48  00 00 00 00  01 84 2b 50
```

The trophy worker is woken by _opd_FUN_004acbc8 (0x004acbc8):

It allocates a 12-byte node from a pool at ctx + 0x17
Writes trophyId into node[2]
Links the node into the queue at ctx + 0x14/0x15
Calls FUN_008f8c58(ctx + 0x34, 0) → sys_cond_signal — wakes the worker
It is called through a vtable slot (OPD entries at 0x017adfd4 / 0x017adff4), with a thunk at _opd_FUN_01190130 that adjusts the this pointer by -8. This is the real IPC path — not FUN_004b404c as previously documented.

Set a breakpoint at 0x004acd44 (bl 0x008f8c58 inside _opd_FUN_004acbc8) and trigger a trophy — r30 at that point will be the trophyId being enqueued.

themain_thread:
```
r0 : 0x41f20900 -> 41 f2 09 18 41 f2 09 18
r1 : 0xd0068630 -> 00 00 00 00 d0 06 86 c0
r2 : 0x1842b50  -> 01 4e 11 40 01 4e 11 60
r3 : 0x41f208e4 -> 00 00 00 02 01 00 00 00
r4 : 0x0
r5 : 0x4
r6 : 0x0
r7 : 0xffffffffffffffff
r8 : 0x41f20918 -> 41 f2 09 00 41 f2 09 00
r9 : 0x1
r10: 0x41f20918 -> 41 f2 09 00 41 f2 09 00
r11: 0x41f20900 -> 41 f2 09 18 41 f2 09 18
r12: 0x24004042
r13: 0x10007be0 -> 00 00 00 00 00 00 00 00
r14: 0x0
r15: 0x0
r16: 0x0
r17: 0x0
r18: 0x0
r19: 0x0
r20: 0x0
r21: 0x0
r22: 0xffffffffffffffff
r23: 0x20000000 -> 00 00 00 00 00 00 00 00
r24: 0x52
r25: 0x5
r26: 0x52
r27: 0x0
r28: 0x41f208dc -> 85 00 a6 00 00 00 00 01
r29: 0x41f208b0 -> 01 7a df a8 42 8a 37 00
r30: 0xc
r31: 0x41f20900 -> 41 f2 09 18 41 f2 09 18
CR: 0x24004042, CR4: [LT=0 GT=1 EQ=0 SO=0]
f0 : 4.29497E+09  [0x41efffffffe00000] (f32=0x4f800000)
f1 : 3.93333      [0x400f7777a0000000] (f32=0x407bbbbd)
f2 : 1            [0x3ff0000000000000] (f32=0x3f800000)
f3 : 0.000310997  [0x3f3461aa80000000] (f32=0x39a30d54)
f4 : -0.054932    [0xbfac200d40000000] (f32=0xbd61006a)
f5 : -1           [0xbff0000000000000] (f32=0xbf800000)
f6 : 1            [0x3ff0000000000000] (f32=0x3f800000)
f7 : 0.0564105    [0x3face1d7c0000000] (f32=0x3d670ebe)
f8 : 0.962102     [0x3feec98a20000000] (f32=0x3f764c51)
f9 : 0.0219723    [0x3f967fe500000000] (f32=0x3cb3ff28)
f10: -0.054932    [0xbfac200d40000000] (f32=0xbd61006a)
f11: 0            [                  ] (f32=0x0)
f12: 0.0299808    [0x3f9eb34760000000] (f32=0x3cf59a3b)
f13: 0            [                  ] (f32=0x0)
f14: 0            [                  ] (f32=0x0)
f15: 0            [                  ] (f32=0x0)
f16: 0            [                  ] (f32=0x0)
f17: 0            [                  ] (f32=0x0)
f18: 0            [                  ] (f32=0x0)
f19: 0            [                  ] (f32=0x0)
f20: 0            [                  ] (f32=0x0)
f21: 0            [                  ] (f32=0x0)
f22: 0            [                  ] (f32=0x0)
f23: 0            [                  ] (f32=0x0)
f24: 0            [                  ] (f32=0x0)
f25: 0            [                  ] (f32=0x0)
f26: 0            [                  ] (f32=0x0)
f27: 0            [                  ] (f32=0x0)
f28: 0            [                  ] (f32=0x0)
f29: 0            [                  ] (f32=0x0)
f30: 0            [                  ] (f32=0x0)
f31: 0.0166667    [0x3f91111120000000] (f32=0x3c888889)
v0 : 4397027c 4293ee2e c3d9984f 40d5d5d2 [x: 302.019 y: 73.9652 z: -435.19 w: 6.68235]
v1 : 3372e47a [x: 5.65528e-08]
v2 : 3a9f32b0 [x: 0.00121458]
v3 : 3f800000 [x: 1]
v4 : 00000000 [x: 0]
v5 : 00000000 [x: 0]
v6 : 00000000 [x: 0]
v7 : 00000000 [x: 0]
v8 : 3f800000 [x: 1]
v9 : 4154668d [x: 13.275]
v10: 00000000 [x: 0]
v11: 4154668d [x: 13.275]
v12: 4154668d [x: 13.275]
v13: 3372e47a [x: 5.65528e-08]
v14: 3db63a9e [x: 0.088979]
v15: be5bbfca [x: -0.214599]
v16: 3f800000 [x: 1]
v17: 40490fdb [x: 3.14159]
v18: 0000001f [x: 4.34403e-44]
v19: 04050607 08090a0b 00010203 0c0d0e0f [x: 1.56368e-36 y: 4.12387e-34 z: 9.25572e-41 w: 1.08665e-31]
v20: 00000000 [x: 0]
v21: 00000000 [x: 0]
v22: 00000000 [x: 0]
v23: 00000000 [x: 0]
v24: 00000000 [x: 0]
v25: 00000000 [x: 0]
v26: 00000000 [x: 0]
v27: 00000000 [x: 0]
v28: 00000000 [x: 0]
v29: 00000000 [x: 0]
v30: 00000000 [x: 0]
v31: 00000000 [x: 0]
CIA: 0x4acd44
LR: 0x4acd00
CTR: 0x4a3f9c
VRSAVE: 0xffffffff
XER: [CA=0 | OV=0 | SO=0 | CNT=0]
VSCR: [SAT=140648099864576 | NJ=1]
FPSCR: [FL=0 | FG=0 | FE=1 | FU=0]
Reservation Addr: none
Reservation Data (entire cache line):
[0x00] 00000000 00000001 00000000 00000001
[0x10] 309fbec0 00000002 00000000 00000001
[0x20] 00000000 00000001 ffffffff 00000005
[0x30] 00023251 00000000 428e6fb0 428e73f0
[0x40] 30903000 00000003 ffffffff ffffffff
[0x50] 309fb080 00000002 ffffffff ffffffff
[0x60] 309da020 00000002 428e7830 428e71d0
[0x70] 00000660 0001eae0 ffffffff ffffffff
```

r30: 0xc which is mantastic voyage trophy!


The complete verified IPC path is:

```
_opd_FUN_004acbc8 (0x004acbc8)
│
│  node = alloc(ctx + 0x17, 0xc bytes)
│  node[2] = param_2 = 0xc  ← trophyId written here
│  link node into queue at ctx + 0x14/0x15
│
│  bl 0x008f8c58  (0x004acd44)  ← r30 = 0xc confirmed at runtime
│       └──► sys_cond_signal(ctx + 0x34)
│                 └──► wakes TrophyManager_WorkerThread
│
▼
TrophyManager_WorkerThread
  dequeue node
  uVar16 = node[2] = 0xc
  sceNpTrophyUnlockTrophy(ctx, hdl, 0xc, &platflag)
```

This supersedes the earlier documented path through FUN_004b404c. The real enqueue-and-signal function is _opd_FUN_004acbc8. You should now repeat this with trophyId = 0x13 (Gone Viral) to confirm the same path fires for that trophy — that would be the final proof that sending event 0x7b through the network stack will ultimately reach _opd_FUN_004acbc8 with param_2 = 0x13 and wake the worker to unlock Gone Viral.

=======

Continue further up the call stack

The vtable base has no code xrefs — it's only ever written into an object at construction time via TOC-relative lis/ori. The call to _opd_FUN_004acbc8 through the vtable cannot be traced statically this way.

The most direct approach from here is the live debugger. Set a breakpoint at 0x004acbc8 and look at LR when it fires — that gives the exact call site that dispatched through the vtable to reach this function.

LR: 0x4a2ef4

====

LR = 0x4a2ef4 — so the call site is at 0x4a2ef0 (LR - 4). Let me look up what function that's in:


















































FUN_004a2e70 is the nearest function start below 0x004a2ef0. Let me decompile it:




This is a vtable dispatch wrapper. It:

Loads a global object via PTR_PTR_0182cd58 (a global singleton pointer)
Calls vtable[0x28] on it to get another object (piVar1)
Calls piVar1->vtable[0x8](piVar1, param_1, param_2) — passing param_1 and param_2 through
The call at 0x004a2ef0 (LR - 4 = 0x4a2ef4 - 4) is inside this function — piVar1->vtable[0x8] resolving to _opd_FUN_004acbc8. So param_1 here maps to the trophy manager ctx and param_2 is the trophyId.

This function is the indirect caller — it routes through a global singleton's vtable to reach the enqueue function. Let me check what calls FUN_004a2e70:


























FUN_004b11d4 is called from 21 places across the codebase. This is the central trophy unlock dispatch function — the single named point that all game systems call to request a trophy unlock. It:

Looks up the trophy definition by a 64-bit hash key (0x9cb9012080f727b9)
Checks an enable flag (*pcVar2 != '\0')
Resolves the trophy ID via FUN_004b10a0(param_1)
Calls FUN_004a2e70(trophyId, param_2) → vtable → _opd_FUN_004acbc8 → sys_cond_signal → wakes worker
This is the function to set your breakpoint on for any trophy unlock. param_2 passed to it is likely a context or flags argument; the trophyId is resolved internally from param_1 (a trophy hash/identifier). Rename this as TrophyManager_RequestUnlock — it's the real public entry point.

=====

i have a breakpoint at FUN_004b11d4, anything to inspect ?

Yes. At the entry of FUN_004b11d4, inspect:

r3 — param_1, the trophy identifier passed in. This is a 64-bit hash or handle that the function uses to look up the trophy definition. Note the value — it will tell us which trophy lookup key maps to Mantastic Voyage, and later which one maps to Gone Viral.

r4 — param_2, the second argument. Unknown purpose so far — could be a context pointer, flags, or player index.

LR — the return address, which tells us which of the 21 callers triggered this for Mantastic Voyage.

Those three values together will tell us what the caller passed in and which code path triggered the unlock.

```
r0 : 0x50091c   -> nop
r1 : 0xd00687f0 -> 00 00 00 00 d0 06 88 b0
r2 : 0x1842b50  -> 01 4e 11 40 01 4e 11 60
r3 : 0x1bd48906e119a954
r4 : 0x0
r5 : 0x0
r6 : 0x2
r7 : 0xffffffffffffffff
r8 : 0x30903000 -> 40 42 d2 f9 6c 4e 4b 8c
r9 : 0x30978ec4 -> 00 00 00 01 00 02 00 02
r10: 0x17fd7a0  -> func(at=0x1189f74, toc=0x1842b50)
r11: 0x2
r12: 0x24004042
r13: 0x10007be0 -> 00 00 00 00 00 00 00 00
r14: 0x0
r15: 0x0
r16: 0x0
r17: 0x0
r18: 0x0
r19: 0x0
r20: 0x0
r21: 0x0
r22: 0xffffffffffffffff
r23: 0x20000000 -> 00 00 00 00 00 00 00 00
r24: 0x55
r25: 0x4
r26: 0x55
r27: 0x30051d20 -> 01 7b 06 b0 00 00 00 00
r28: 0x30073f20 -> 01 79 6e 48 00 00 00 04
r29: 0x3002f7c0 -> 01 79 6b d0 00 00 00 0d
r30: 0x3061e290 -> 30 32 02 a0 30 02 7b e0
r31: 0x0
CR: 0x24004042, CR7: [LT=0 GT=0 EQ=1 SO=0]
f0 : 100          [0x4059000000000000] (f32=0x42c80000)
f1 : 2.83333      [0x4006aaaac0000000] (f32=0x40355556)
f2 : 1            [0x3ff0000000000000] (f32=0x3f800000)
f3 : 0            [                  ] (f32=0x0)
f4 : 0            [                  ] (f32=0x0)
f5 : -1           [0xbff0000000000000] (f32=0xbf800000)
f6 : 1            [0x3ff0000000000000] (f32=0x3f800000)
f7 : -0.000213569 [0xbf2bfe3000000000] (f32=0xb95ff180)
f8 : 0.131874     [0x3fc0e14340000000] (f32=0x3e070a1a)
f9 : -0.000217014 [0xbf2c71c6a0000000] (f32=0xb9638e35)
f10: 0            [                  ] (f32=0x0)
f11: 0            [                  ] (f32=0x0)
f12: 0.0513206    [0x3faa46b220000000] (f32=0x3d523591)
f13: 0            [                  ] (f32=0x0)
f14: 0            [                  ] (f32=0x0)
f15: 0            [                  ] (f32=0x0)
f16: 0            [                  ] (f32=0x0)
f17: 0            [                  ] (f32=0x0)
f18: 0            [                  ] (f32=0x0)
f19: 0            [                  ] (f32=0x0)
f20: 0            [                  ] (f32=0x0)
f21: 0            [                  ] (f32=0x0)
f22: 0            [                  ] (f32=0x0)
f23: 0            [                  ] (f32=0x0)
f24: 0            [                  ] (f32=0x0)
f25: 0            [                  ] (f32=0x0)
f26: 0            [                  ] (f32=0x0)
f27: 0            [                  ] (f32=0x0)
f28: 0            [                  ] (f32=0x0)
f29: 0            [                  ] (f32=0x0)
f30: 0            [                  ] (f32=0x0)
f31: 0.0166667    [0x3f91111120000000] (f32=0x3c888889)
v0 : 439915fe 4293dfcf c3d726bb 3e02aa54 [x: 306.172 y: 73.9371 z: -430.303 w: 0.127603]
v1 : 3f29dd33 [x: 0.663531]
v2 : 3bc43ea8 [x: 0.00598891]
v3 : 3f800000 [x: 1]
v4 : 00000000 [x: 0]
v5 : 00000000 [x: 0]
v6 : 00000000 [x: 0]
v7 : 3e9d05a5 bd8500d2 3f289a45 7fc00000 [x: 0.306684 y: -0.064943 z: 0.658604 w: nan]
v8 : 3c3f2163 [x: 0.0116657]
v9 : bf680c73 00000000 3ed83c3d 00000000 [x: -0.90644 y: 0 z: 0.422335 w: 0]
v10: bd84ecfe [x: -0.0649052]
v11: 3e000f0e 00000000 be896c7c 00000000 [x: 0.125057 y: 0 z: -0.268406 w: 0]
v12: 3e979bb6 [x: 0.29611]
v13: 3e000f0e bd84ecfe be896c7c 00000000 [x: 0.125057 y: -0.0649052 z: -0.268406 w: 0]
v14: 3db63a9e [x: 0.088979]
v15: be5bbfca [x: -0.214599]
v16: 3f800000 [x: 1]
v17: 40490fdb [x: 3.14159]
v18: 0000001f [x: 4.34403e-44]
v19: 04050607 08090a0b 00010203 0c0d0e0f [x: 1.56368e-36 y: 4.12387e-34 z: 9.25572e-41 w: 1.08665e-31]
v20: 00000000 [x: 0]
v21: 00000000 [x: 0]
v22: 00000000 [x: 0]
v23: 00000000 [x: 0]
v24: 00000000 [x: 0]
v25: 00000000 [x: 0]
v26: 00000000 [x: 0]
v27: 00000000 [x: 0]
v28: 00000000 [x: 0]
v29: 00000000 [x: 0]
v30: 00000000 [x: 0]
v31: 00000000 [x: 0]
CIA: 0x4b11d4
LR: 0x500948
CTR: 0x118d054
VRSAVE: 0xffffffff
XER: [CA=0 | OV=0 | SO=0 | CNT=0]
VSCR: [SAT=140648099864576 | NJ=1]
FPSCR: [FL=1 | FG=0 | FE=0 | FU=0]
Reservation Addr: none
Reservation Data (entire cache line):
[0x00] 00000000 00000001 00000000 00000001
[0x10] 309fbec0 00000002 ffffffff ffffffff
[0x20] 00000000 00000001 ffffffff 00000004
[0x30] 00041d27 00000000 428dd160 428e08a0
[0x40] 30903000 00000002 ffffffff ffffffff
[0x50] 309fb080 00000002 ffffffff ffffffff
[0x60] 309da020 00000002 428f18a0 428d7600
[0x70] 00000550 0001eae0 ffffffff ffffffff
```

=====


The vtable base at 0x017adfd4 has no code xrefs — it's loaded via TOC-relative addressing. Let me find what calls the thunk directly and also what other functions are adjacent in the vtable to understand the object type:






0x017adfe0 → TrophyManager_WorkerThread entry point — confirmed this is the trophy manager vtable. Now let me look at the slot just before our function (0x017adfd8 → FUN_004a3f9c) — that's likely the dispatch function that calls into _opd_FUN_004acbc8 through the vtable:







_opd_FUN_004a3f9c is the bitset query — checks if a trophy is already unlocked (param_2 < 0x80, reads from param_1 + 0x18 bitset). This is the vtable gate called at 0x004acc24 inside _opd_FUN_004acbc8 — it is (*param_1->vtable[0x30])(param_1, trophyId), the pre-enqueue validator that returns 0 if the trophy is already in the local unlock bitset.

Now let me find what actually calls _opd_FUN_004acbc8 through the vtable — look for code that loads the vtable and calls at the slot offset for 0x017adfd4:

The slot 0x017adfd4 is at offset +0x14 from 0x017adfc0 (the vtable base). So callers do *(vtable + 0x14)(obj, trophyId). Let me search for callers of TrophyQueue_PushUnlockRequest and Trophy_DispatchUnlockById to trace what feeds into the vtable call:








TrophyQueue_PushUnlockRequest writes into the ring buffer at param_1 + 0x20 — but from the runtime debugging we now know the worker actually reads from a different queue (ctx + 0x14) populated by _opd_FUN_004acbc8. These are two separate queues on the same object. Let me find what calls _opd_FUN_004acbc8 through the vtable by looking at what function dispatches through *(vtable + 0x14):







That's another ring buffer writer variant. The vtable dispatch must come from elsewhere. Let me look directly for code that calls through vtable offset +0x14 on the trophy manager object — find callers that do (*(vtable+0x14))(obj, trophyId):











The vtable call *(param_1 + 300)->vtable[0x4c] — that's +0x4c offset into a vtable. This is dispatching into a subscriber/listener system, not the trophy manager directly. Let me take a more direct approach and look at what the runtime register dump told us — r29 = 0x41f208b0 is param_1 (ctx base). Find what calls _opd_FUN_004acbc8 by searching for code that loads the vtable from that object type and calls slot +0x14:






The vtable base has no code xrefs — it's only ever written into an object at construction time via TOC-relative lis/ori. The call to _opd_FUN_004acbc8 through the vtable cannot be traced statically this way.

The most direct approach from here is the live debugger. Set a breakpoint at 0x004acbc8 and look at LR when it fires — that gives the exact call site that dispatched through the vtable to reach this function.


functions Trophy_DispatchUnlockById, TrophyQueue_PushUnlockRequest and TrophyManager_OnTrophyUnlock are named by you right and not in this session? are you confident these are still named correctly, after our latest findings?

Good question. Let me re-examine each one in light of what we now know.











Honest assessment of each:

Trophy_DispatchUnlockById — name is misleading
It just calls TrophyQueue_PushUnlockRequest() with no visible arguments in the decompiler (TOC-only wrapper). The name implies it dispatches by ID directly to an unlock — but we now know this feeds the ring buffer at ctx + 0x20, not the queue at ctx + 0x14 that actually wakes the worker. We haven't confirmed this path reaches _opd_FUN_004acbc8 at all. The name overstates what we know.

TrophyQueue_PushUnlockRequest — name is probably wrong
It writes into the ring buffer at ctx + 0x20. But the real enqueue that wakes the worker is _opd_FUN_004acbc8, which writes into the linked list at ctx + 0x14. We have no runtime proof that TrophyQueue_PushUnlockRequest leads to a trophy unlock. It may be a different subsystem entirely — perhaps an achievement tracker or a UI notification queue — not the trophy unlock queue. The name was assigned by inference before the runtime debugging revealed _opd_FUN_004acbc8 as the real path.

TrophyManager_OnTrophyUnlockedCallback — name is plausible but unconfirmed
It is called by TrophyManager_WorkerThread after sceNpTrophyUnlockTrophy succeeds, allocates an event object, and posts it to the engine event bus. This behaviour matches a post-unlock notification callback. The name is reasonable but was inferred from static analysis only — we haven't set a breakpoint on it at runtime to confirm it fires when a trophy pops.

Summary
Function	Confidence in name
Trophy_DispatchUnlockById	Low — misleading, unconfirmed path
TrophyQueue_PushUnlockRequest	Low — writes to a different queue than what wakes the worker
TrophyManager_OnTrophyUnlockedCallback	Medium — plausible but unconfirmed at runtime
The one function we have confirmed at runtime is _opd_FUN_004acbc8 — that is the real enqueue + signal function. Everything else in the old naming should be treated as unverified inference.


can you rename these 3 functions back to how Ghidra would call them?






Done. All three are back to Ghidra's default names:

Trophy_DispatchUnlockById → FUN_00f2deb0
TrophyQueue_PushUnlockRequest → FUN_00cadc64
TrophyManager_OnTrophyUnlockedCallback → FUN_004bd968
The only function we've confirmed at runtime with a proper name is TrophyManager_WorkerThread (0x004bdb60) and _opd_FUN_004acbc8 (the real enqueue + signal, pending a rename decision from you).


back to the last breakpoint you mentioend, i see LR: 0x4a2ef4



LR = 0x4a2ef4 — so the call site is at 0x4a2ef0 (LR - 4). Let me look up what function that's in:


















































FUN_004a2e70 is the nearest function start below 0x004a2ef0. Let me decompile it:




This is a vtable dispatch wrapper. It:

Loads a global object via PTR_PTR_0182cd58 (a global singleton pointer)
Calls vtable[0x28] on it to get another object (piVar1)
Calls piVar1->vtable[0x8](piVar1, param_1, param_2) — passing param_1 and param_2 through
The call at 0x004a2ef0 (LR - 4 = 0x4a2ef4 - 4) is inside this function — piVar1->vtable[0x8] resolving to _opd_FUN_004acbc8. So param_1 here maps to the trophy manager ctx and param_2 is the trophyId.

This function is the indirect caller — it routes through a global singleton's vtable to reach the enqueue function. Let me check what calls FUN_004a2e70:


























FUN_004b11d4 is called from 21 places across the codebase. This is the central trophy unlock dispatch function — the single named point that all game systems call to request a trophy unlock. It:

Looks up the trophy definition by a 64-bit hash key (0x9cb9012080f727b9)
Checks an enable flag (*pcVar2 != '\0')
Resolves the trophy ID via FUN_004b10a0(param_1)
Calls FUN_004a2e70(trophyId, param_2) → vtable → _opd_FUN_004acbc8 → sys_cond_signal → wakes worker
This is the function to set your breakpoint on for any trophy unlock. param_2 passed to it is likely a context or flags argument; the trophyId is resolved internally from param_1 (a trophy hash/identifier). Rename this as TrophyManager_RequestUnlock — it's the real public entry point.


i have a breakpoint at FUN_004b11d4, anything to inspect ?

Yes. At the entry of FUN_004b11d4, inspect:

r3 — param_1, the trophy identifier passed in. This is a 64-bit hash or handle that the function uses to look up the trophy definition. Note the value — it will tell us which trophy lookup key maps to Mantastic Voyage, and later which one maps to Gone Viral.

r4 — param_2, the second argument. Unknown purpose so far — could be a context pointer, flags, or player index.

LR — the return address, which tells us which of the 21 callers triggered this for Mantastic Voyage.

Those three values together will tell us what the caller passed in and which code path triggered the unlock.




Key values:

r3 = 0x1bd48906e119a954 — the 64-bit trophy lookup hash for Mantastic Voyage. This is what the game uses internally to identify this trophy before resolving it to the numeric ID 0xc.
r4 = 0x0 — second argument is zero, likely unused or a flags/context field that isn't needed for this call path.
LR = 0x500948 — caller is at 0x500944 (LR - 4). Let me look that up:




Found it. The caller is FUN_00500734 — and it contains the exact hash 0x1bd48906e119a954 for Mantastic Voyage. Look at the condition:

if (('\x02' < *(char *)((int)piVar3 + 0x167e)) &&
   (cVar5 = FUN_004b12ec(0x1bd48906e119a954), cVar5 == '\0')) {
    skate3_themain_thread_trigger_trophy_parent(0x1bd48906e119a954, 0);
}

This reads:

*(char *)(piVar3 + 0x167e) > 2 — the manual count must be greater than 2 (i.e. at least 3 manuals in the line)
FUN_004b12ec(hash) returns '\0' (trophy not yet unlocked) — don't re-award if already earned
If both pass → call FUN_004b11d4 with the Mantastic Voyage hash
The function also checks other trophies in the same block using their hashes:

0xb598d5bd5d16ec4f — score > 9999 trophy
0x2a9d5f12bc41ee19 — score > 29999 trophy
0xe9c379e93fa2693f — trick count > 2 trophy
0xd127d95b8d349b72 — some float threshold trophy
0x1bd48906e119a954 — Mantastic Voyage (3+ manuals)
This is the trick line evaluation function — it runs after completing a trick line and checks all trick-line-related trophies. piVar3 + 0x167e is the manual counter in the player/trick state struct.

Also notable: Ghidra has already renamed FUN_004b11d4 to skate3_themain_thread_trigger_trophy_parent from our earlier work. This confirms FUN_004b11d4 is the correct single entry point for all trophy unlock requests.

------

ghidra pseudo code snippet that triggers mantastic voyage trophy. Confirmed with debugger breakpoint:
```cpp
if (('\x02' < *(char *)((int)piVar3 + 0x167e)) &&
    (cVar5 = FUN_004b12ec(0x1bd48906e119a954), cVar5 == '\0')) {
  skate3_themain_thread_trigger_trophy_hash(0x1bd48906e119a954,0);
}
```
