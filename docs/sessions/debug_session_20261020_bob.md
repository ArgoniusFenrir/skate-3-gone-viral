# Debug Session 20261020 — Trophy Unlock IPC Investigation

**Binary:** Skate 3 PS3 `v1.05` decrypted ELF
**Tool:** RPCS3 debugger + Ghidra via GhidraMCP
**Goal:** Confirm at runtime how the trophy worker thread gets woken up and trace the
complete call chain from game logic to `sceNpTrophyUnlockTrophy`.

---

## 1. Starting Point & Corrections to Prior Analysis

Prior static analysis named three functions that were later found to be incorrect or
unconfirmed:

| Old name | Address | Status |
|---|---|---|
| `Trophy_DispatchUnlockById` | `0x00f2deb0` | Reverted — unconfirmed path |
| `TrophyQueue_PushUnlockRequest` | `0x00cadc64` | Reverted — writes to a different queue than what wakes the worker |
| `TrophyManager_OnTrophyUnlockedCallback` | `0x004bd968` | Reverted — plausible but unconfirmed at runtime |

All three were renamed back to Ghidra defaults (`FUN_00f2deb0`, `FUN_00cadc64`,
`FUN_004bd968`). The ring buffer at `ctx + 0x20` written by `FUN_00cadc64` is a
separate structure from the linked list at `ctx + 0x14` that actually feeds the
worker thread — these were incorrectly assumed to be the same queue.

---

## 2. Confirming the Worker Thread Wakeup Mechanism

### Worker sleeps at `0x008fa51c`

The trophy worker thread (`TrophyManager_WorkerThread` @ `0x004bdb60`) was observed
in the RPCS3 debugger sleeping at `0x008fa51c` — the `sc` instruction for
`sys_cond_wait` (syscall `0x6b`) inside `FUN_008fa40c`.

### Confirmed: woken by signal, not timeout

Breakpoint set at `0x008fa520` (first instruction after the `sys_cond_wait` `sc`).
When the worker woke:

- **`r3 = 0x0`** — `sys_cond_wait` return code 0 = woken by `sys_cond_signal`, not a
  timeout

This disproves the earlier hypothesis that the worker polls on a timer.

### Finding the condvar handle

With the worker paused at `0x008fa51c`:
- `r28 = 0x41f208e4` — base of the condvar struct
- Condvar LV2 handle = `*(r28 + 0x8)` = `*(0x41f208ec)` — read from RPCS3 memory
  viewer

### Finding the signal caller

All `sys_cond_signal` calls in the binary go through `FUN_008f8c58`. Breakpoint set
at `0x008f8cd8` (the `syscall_sys_cond_signal` call inside `FUN_008f8c58`).

When the breakpoint fired with `r3` matching the trophy worker's condvar handle:
- **`LR = 0x4acd48`** — return address into `_opd_FUN_004acbc8`
- **`r2 = 0x1842b50`** — TOC base of the calling thread

The call site is at `0x4acd44` (`bl 0x008f8c58`) inside `_opd_FUN_004acbc8`.

---

## 3. The Real Enqueue + Signal Function: `_opd_FUN_004acbc8`

**Address:** `0x004acbc8`
**Confirmed at runtime** as the function that enqueues a trophy ID and wakes the
worker.

### Confirmed via two breakpoints

| Breakpoint | Register | Value | Meaning |
|---|---|---|---|
| `0x004acbc8` (function entry) | `r4` | `0xc` | `param_2` = trophyId = Mantastic Voyage |
| `0x004acd44` (signal call) | `r30` | `0xc` | trophyId still `0xc` at point of signal |

### What the function does

```c
undefined4 _opd_FUN_004acbc8(int *param_1, int param_2 /*trophyId*/)
{
    // 1. Shutdown guard: ctx+0x5948 — drop if manager shutting down
    if (*(char *)(param_1 + 0x5948) == '\0') return 0;

    // 2. Pre-enqueue gate: vtable call (*param_1->vtable[0x30])(param_1, trophyId)
    //    Returns 0 if trophy already in local unlock bitset → drop
    if (vtable[0x30](param_1, param_2) != 0) return 0;

    // 3. Lock queue mutex at ctx+0xb
    sys_mutex_lock(param_1 + 0xb);

    // 4. Dedup scan: walk pending queue at ctx+0x14
    //    If trophyId already queued → skip enqueue, just unlock and return
    if (already_queued(param_1 + 0x14, param_2)) {
        sys_mutex_unlock(param_1 + 0xb);
        return 1;
    }

    // 5. Allocate 12-byte node from pool at ctx+0x17
    node = alloc(param_1 + 0x17, 0xc, 4, 0);
    node[2] = param_2;              // ← trophyId stored at word [2]

    // 6. Link node into tail of queue at ctx+0x14/0x15, increment count at ctx+0x16

    // 7. Signal the worker
    sys_cond_signal(param_1 + 0x34); // ← wakes TrophyManager_WorkerThread

    // 8. Unlock mutex
    sys_mutex_unlock(param_1 + 0xb);
    return 1;
}
```

### Key struct offsets on the trophy manager object

| Offset | Purpose |
|---|---|
| `+0x5948` | Shutdown flag |
| `+0xb` | Queue mutex |
| `+0x14` / `+0x15` | Pending trophy linked list (head / tail) |
| `+0x16` | Queue entry count |
| `+0x17` | Node memory pool |
| `+0x34` | Condition variable (signalled to wake worker) |

### Dedup behaviour

If the same `trophyId` is already in the pending queue, the enqueue is skipped
entirely. Gone Viral (`0x13`) can only appear once in the queue at any time.

### Worker reads `node[2]` as trophyId

`TrophyManager_WorkerThread` dequeues the node and reads:
```
004bdf48: lwz r31, 0x8(r8)    ← r31 = node[2] = trophyId
```
This value is passed directly as `r5` to `sceNpTrophyUnlockTrophy`.

---

## 4. Call Chain Above `_opd_FUN_004acbc8`

### Vtable thunk

`_opd_FUN_004acbc8` is registered in the trophy manager vtable at `0x017adfd4`
(`+0x14` from vtable base `0x017adfc0`). It is called through a thunk:

```c
// _opd_FUN_01190130 @ 0x01190130
void _opd_FUN_01190130(longlong param_1) {
    _opd_FUN_004acbc8(param_1 + -8);  // adjust this ptr by -8
}
```

### Indirect dispatch wrapper

`FUN_004a2e70` (`0x004a2e70`) is the intermediate dispatcher:

```c
undefined1 FUN_004a2e70(undefined4 param_1, undefined4 param_2) {
    piVar1 = (*global_singleton->vtable[0x28])(global_singleton); // get trophy mgr
    if (piVar1 != NULL)
        return piVar1->vtable[0x8](piVar1, param_1, param_2);     // → _opd_FUN_004acbc8
    return 0;
}
```

### Central trophy request function

`FUN_004b11d4` (`skate3_themain_thread_trigger_trophy_parent`) is called from **21
places** across the codebase. It is the single public entry point for all trophy
unlock requests:

```c
undefined1 FUN_004b11d4(undefined8 param_1 /*trophy_hash*/, undefined4 param_2) {
    // Look up trophy definition by 64-bit hash
    // Check enable flag
    // Resolve numeric trophyId via FUN_004b10a0
    // Call FUN_004a2e70(trophyId, param_2)
}
```

### Trick line evaluator caller

`FUN_00500734` (`0x00500734`) is the trick line evaluation function that calls
`FUN_004b11d4` for Mantastic Voyage. Confirmed via `LR = 0x500948` at breakpoint.

It checks `*(char *)(piVar3 + 0x167e) > 2` — the manual count in the player/trick
state struct must exceed 2 (i.e. 3+ manuals in a line) before dispatching:

```c
if (('\x02' < *(char *)((int)piVar3 + 0x167e)) &&
   (FUN_004b12ec(0x1bd48906e119a954) == '\0')) {
    skate3_themain_thread_trigger_trophy_parent(0x1bd48906e119a954, 0);
}
```

Other trophies evaluated in the same function:
- `0xb598d5bd5d16ec4f` — score > 9,999
- `0x2a9d5f12bc41ee19` — score > 29,999
- `0xe9c379e93fa2693f` — trick count > 2
- `0xd127d95b8d349b72` — float threshold (grind length / speed)
- `0x1bd48906e119a954` — **Mantastic Voyage** (3+ manuals)

---

## 5. Confirmed Trophy Hash

| Trophy | Hash (`r3` at `FUN_004b11d4`) | Numeric ID |
|---|---|---|
| Mantastic Voyage | `0x1bd48906e119a954` | `0x0c` (12) |
| Gone Viral | unknown — not yet triggered | `0x13` (19) |

---

## 6. Complete Verified Call Chain (Mantastic Voyage)

```
FUN_00500734 / skate3_evaluate_trick_line
  │  condition: player manual count at struct+0x167e > 2
  │  hash: 0x1bd48906e119a954
  │  r3 = 0x1bd48906e119a954  (confirmed at breakpoint)
  │  r4 = 0x0
  │
  ▼
FUN_004b11d4 / skate3_themain_thread_trigger_trophy_parent
  │  r3 = 0x1bd48906e119a954  (confirmed at breakpoint)
  │  r4 = 0x0
  │  resolves hash → trophyId = 0xc
  │
  ▼
FUN_004a2e70
  │  routes through global singleton vtable
  │
  ▼
_opd_FUN_004acbc8 (0x004acbc8)
  │  r4 = 0xc  (confirmed at breakpoint: param_2 = trophyId)
  │  allocates node, writes node[2] = 0xc
  │  r30 = 0xc at signal call (confirmed at 0x004acd44)
  │  sys_cond_signal(ctx + 0x34)
  │
  ▼
TrophyManager_WorkerThread (0x004bdb60)
  │  r3 = 0x0 at 0x008fa520 (confirmed: woken by signal)
  │  dequeues node, reads uVar16 = node[2] = 0xc
  │
  ▼
sceNpTrophyUnlockTrophy(ctx+0x10, ctx+0x14, 0xc, &platflag)
  │  r5 = 0xc  (confirmed at 0x0144cec4 breakpoint)
  │
  ▼
  Trophy unlocked in PS3 system
```

---

## 7. Ghidra Annotations Added

Disassembly comments set on `_opd_FUN_004acbc8` at:

| Address | Comment |
|---|---|
| `0x004acbc8` | Function purpose, vtable call site, runtime proof |
| `0x004acbe4` | Shutdown guard (`ctx + 0x5948`) |
| `0x004acc24` | Vtable pre-enqueue gate |
| `0x004acc58` | Mutex lock |
| `0x004acc7c` | Dedup scan |
| `0x004accfc` | Node allocation |
| `0x004acd1c` | `node[2] = trophyId` — payload the worker reads |
| `0x004acd24` | Node linked into queue, count incremented |
| `0x004acd44` | `sys_cond_signal(ctx + 0x34)` — wakes worker |
| `0x004acd4c` | Mutex unlock after signal |
