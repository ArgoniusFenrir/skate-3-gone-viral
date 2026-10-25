# TrophyManager Worker Thread — Lifecycle & IPC

**Binary:** Skate 3 PS3 `v1.05` decrypted ELF  
**Analysis method:** Ghidra decompilation via GhidraMCP (live session)

---

## 1. Thread Entry Point & Startup

The worker thread is launched by the engine's generic thread-creation wrapper chain.
The entry point `0x004bdb60` (`TrophyManager_WorkerThread`) is not called directly — it
is stored as a function pointer in the OPD (Object Procedure Descriptor) table at
`0x017ff1a0`, pointed to by a data slot at `0x017adfe0`.

### Creation call chain

```
FUN_008f9bd0 / FUN_008f9bf0          thin wrappers (joinable = 0 or 1)
        └──► FUN_008f9934            allocates thread context struct,
                                     reads entry point from TOC slot puVar1[0x1eea]
                    └──► sys_ppu_thread_create(
                               threadCtx,          // arg passed to entry point
                               entryPoint,         // _opd_FUN_011a7230
                               arg = threadCtx,
                               priority,           // from manager config
                               stackSize = 0x10000,
                               flags = 0,
                               name)
```

The registered entry point is a one-line OPD thunk at `0x011a7230`:

```c
// _opd_FUN_011a7230 @ 0x011a7230
void _opd_FUN_011a7230(longlong param_1) {
    TrophyManager_WorkerThread(param_1 + -0xc);
}
```

The `-0xc` offset adjusts the pointer passed by `sys_ppu_thread_create` back to the
base of the TrophyManager object (`param_1`), which is used throughout the worker thread.

### Startup sequence (inside the worker thread)

On first entry, before entering the dispatch loop, the thread performs one-time
initialization:

1. `sceNpTrophyRegisterContext(ctx, hdl, callback_ptr, 0, 0)` — registers the NP
   trophy context with the PS3 SDK.
2. `sceNpTrophyGetTrophyUnlockState(ctx, hdl, ...)` — queries which trophies are
   already unlocked on the current profile.
3. Builds a local 128-bit unlock bitset at `param_1 + 0x18` — one bit per trophy ID —
   from the query result.
4. `sceNpTrophyGetGameInfo` + per-trophy `sceNpTrophyGetTrophyInfo` loop — loads
   trophy metadata. Exits early if the shutdown flag at `param_1 + 0x5949` is set.
5. Sets the ready flag at `param_1 + 0x594b = 1` to signal that initialization is
   complete, then falls through into the main dispatch loop.

---

## 2. Main Dispatch Loop & Sleep Mechanism

After initialization the thread enters an infinite loop that drains the unlock queue.

### Pseudocode

```c
// Main loop — TrophyManager_WorkerThread @ 0x004bdb60
int *queue_head = (int *)(param_1 + 0x50);   // intrusive linked list head

while (true) {
    // Check shutdown flag
    if (*(char *)(param_1 + 0x5948) == 0) {
        *(char *)(param_1 + 0x5948) = 0;
        return 0;
    }

    // Lock the queue mutex
    FUN_008fa300(param_1 + 0x34, ...);        // sys_mutex_lock

    // If queue is empty, release mutex and block on condition variable
    if (*queue_head == (int)queue_head) {
        FUN_008fa40c(param_1 + 0x34, ...);    // sys_cond_wait  ← thread sleeps here
        // re-check loop on wake
        continue;
    }

    // Dequeue one entry
    int *entry   = (int *)*queue_head;
    uint trophyId = entry[2];                 // the queued trophyId (e.g. 0x13)
    // unlink entry from list, return node to free pool at param_1 + 0x5c

    FUN_008f8dd0(...);                        // sys_mutex_unlock

    if ((int)trophyId < 0) continue;         // sentinel / invalid guard

    // Unlock the trophy via PS3 SDK
    int platflag = -1;
    int rc = sceNpTrophyUnlockTrophy(
                 *(int *)(param_1 + 0x10),   // context
                 *(int *)(param_1 + 0x14),   // handle
                 trophyId,
                 &platflag);

    if (rc >= 0) {
        // Mark bit in local unlock bitset at param_1 + 0x18
        puVar2[trophyId >> 6] |= (1ULL << (trophyId & 0x3f));
        // Notify engine listeners
        TrophyManager_OnTrophyUnlockedCallback(param_1, trophyId);
    }
}
```

### Synchronization primitives used

| Primitive | PS3 LV2 syscall | Location in manager struct | Purpose |
|---|---|---|---|
| Mutex | `sys_mutex_lock` / `sys_mutex_unlock` | `param_1 + 0x34` | Protects queue dequeue |
| Condition variable | `sys_cond_wait` / `sys_cond_signal` | `param_1 + 0x34` (lazy-created) | Wakes worker when queue is non-empty |

The condition variable is created lazily inside `FUN_008fa40c` via
`syscall_sys_cond_create` on the first call if it does not already exist.

### Shutdown flags

| Address | Meaning |
|---|---|
| `param_1 + 0x5948` | Main shutdown flag — set externally to request thread exit; worker returns `0` on next loop iteration |
| `param_1 + 0x5949` | Init-phase early-exit flag — checked during the per-trophy metadata loop at startup |
| `param_1 + 0x594b` | Ready flag — set to `1` when initialization is complete |

---

## 3. IPC — How Other Threads Enqueue Trophy Requests

Any game thread that wants to unlock a trophy calls into this chain:

```
Trophy_DispatchUnlockById(ctx, trophyId)            FUN_00f2deb0
        └──► TrophyQueue_PushUnlockRequest(ctx, trophyId)  FUN_00cadc64
                  │  1. Check shutdown flag at ctx + 0x83; return if set
                  │  2. FUN_00cabf9c(ctx) — overflow guard; sets a flag in the
                  │     queue's page table if both fill-level watermarks are exceeded
                  │  3. Write 5-word entry into ring buffer at ctx + 0x20:
                  │       [trophyId, 0xffffffff, type, 0, 2]
                  │  4. Advance write pointer
                  └──► FUN_004b404c(ctx, param2, param3, param4)
                              │  enqueue into worker FIFO at ctx + 0x50
                              └──► FUN_00818194 / FUN_00817b5c
                                        └──► sys_cond_signal  ← wakes worker thread
```

### Queue entry layout (5 words at `ctx + 0x20`)

| Word offset | Value | Meaning |
|---|---|---|
| `+0` | `trophyId` (`param_2`) | Trophy ID written to ring buffer slot 0; becomes list link pointer on insert (overwritten) |
| `+1` | `0xffffffff` | Unused / sentinel; becomes list prev pointer on insert (overwritten) |
| `+2` | `local_38` = `param_2` | **Trophy ID written to payload slot; survives dequeue intact — this is what the worker reads** |
| `+3` | `0` | Reserved |
| `+4` | `2` | Event class / priority |

> **Why word `[2]` carries the trophy ID:** `local_38 = (uint)local_38._2_2_` is
> Ghidra's big-endian representation of the lower 16 bits of `param_2` (register
> `r4`). For `param_2 = 0x13` this evaluates to `0x0013 = 0x13`. Words `[0]` and
> `[1]` are overwritten by list link pointers during insertion into the intrusive
> linked list, so the payload that survives to the worker is at `[2]`.

### Two data structures

The TrophyManager object maintains two related structures that together form the IPC
path from push to unlock:

| Offset | Role |
|---|---|
| `ctx + 0x20` | **Incoming ring buffer write pointer** — advanced by `TrophyQueue_PushUnlockRequest` |
| `ctx + 0x50` | **Worker FIFO intrusive linked list head** — consumed by `TrophyManager_WorkerThread` |

`FUN_004b404c` transfers entries from the ring buffer into the linked list and signals
the condition variable to wake the worker.

---

## 4. Proof: `TrophyQueue_PushUnlockRequest` → `TrophyManager_WorkerThread`

This section proves, field-by-field from the Ghidra decompilation, that the trophy ID
written by `TrophyQueue_PushUnlockRequest` is the exact value passed as `trophyId` to
`sceNpTrophyUnlockTrophy` inside `TrophyManager_WorkerThread`.

### Step 1 — Push writes `trophyId` into ring buffer word `[2]`

```c
// TrophyQueue_PushUnlockRequest @ 0x00cadc64
puVar1 = *(undefined4 **)(param_1 + 0x20);    // load ring buffer write pointer from ctx+0x20
*(undefined4 **)(param_1 + 0x20) = puVar1 + 5; // advance write pointer by 5 words
*puVar1     = param_2;        // word [0] = trophyId  (e.g. 0x13)
puVar1[1]   = 0xffffffff;     // word [1] = sentinel
puVar1[2]   = local_38;       // word [2] = local_38 = (uint)param_2 = trophyId  ← KEY
puVar1[3]   = 0;              // word [3] = reserved
puVar1[4]   = 2;              // word [4] = event class
```

`local_38` is the lower 16 bits of `param_2` (`r4`) in big-endian PPC — for
`param_2 = 0x13`, `local_38 = 0x13`.

### Step 2 — Worker startup zeros `ctx + 0x20` (same field)

```c
// TrophyManager_WorkerThread @ 0x004bdb60 — startup
*(undefined8 *)(param_1 + 0x20) = 0;   // initialises ring buffer to empty
```

Both functions operate on `param_1 + 0x20` — the push function writes entries, the
worker initialises the field to zero on startup. This is the same shared object field.

### Step 3 — Worker reads `trophyId` from linked list node word `[2]`

```c
// TrophyManager_WorkerThread @ 0x004bdb60 — dispatch loop
piVar11 = (int *)(param_1 + 0x50);  // linked list head sentinel at ctx+0x50

piVar14 = (int *)*piVar11;           // first node pointer
if (piVar14 == piVar11) {            // list empty → sys_cond_wait
    FUN_008fa40c(...);
    piVar14 = (int *)*piVar11;
}
if (piVar11 == piVar14) break;       // still empty after wake → loop

piVar5 = (int *)piVar14[1];          // next-node pointer (list mechanics)
uVar16 = piVar14[2];                 // ← trophyId = node word [2]  ← KEY

// unlink the node
*piVar5 = *piVar14;
*(int **)(*piVar14 + 4) = piVar5;
...
if ((int)uVar16 < 0) goto LAB_004bdef8;  // guard: skip if sentinel/invalid

sceNpTrophyUnlockTrophy(
    *(int *)(param_1 + 0x10),   // NP context
    *(int *)(param_1 + 0x14),   // NP handle
    uVar16,                      // ← the trophyId read from node[2]
    &local_b44);
```

### Step 4 — Node word `[2]` is ring buffer word `[2]`

`FUN_004b404c` transfers the 5-word ring buffer entry into the linked list. Words `[0]`
and `[1]` are overwritten by the intrusive list link pointers during insertion.
Word `[2]` (and beyond) is payload — it is copied verbatim into the node and is never
modified before the worker reads it.

| Ring buffer word | Written by push | After list insertion | Worker reads as |
|---|---|---|---|
| `[0]` | `param_2` (`0x13`) | overwritten by list `next` ptr | — |
| `[1]` | `0xffffffff` | overwritten by list `prev` ptr | — |
| **`[2]`** | **`local_38` = `0x13`** | **unchanged** | **`uVar16` = `0x13`** |
| `[3]` | `0` | unchanged | — |
| `[4]` | `2` | unchanged | — |

### Complete proof chain

```
TrophyQueue_PushUnlockRequest(ctx, 0x13)
│
│  puVar1 = *(ctx + 0x20)            // read ring buffer write pointer
│  puVar1[2] = local_38 = 0x13      // write trophyId to payload slot
│  advance ctx + 0x20 by 5 words
│
▼
FUN_004b404c / FUN_00818194
│
│  transfer 5-word entry from ring buffer (ctx+0x20) into linked list (ctx+0x50)
│  words [0],[1] overwritten by list link pointers
│  word [2] = 0x13 survives intact
│  sys_cond_signal → wakes TrophyManager_WorkerThread
│
▼
TrophyManager_WorkerThread
│
│  piVar11 = ctx + 0x50              // linked list head
│  piVar14 = *piVar11                // dequeue first node
│  uVar16  = piVar14[2]              // ← reads 0x13 from payload word [2]
│  guard: (int)uVar16 < 0 → false   // 0x13 passes
│
▼
sceNpTrophyUnlockTrophy(
    ctx+0x10,   // NP context handle
    ctx+0x14,   // NP request handle
    0x13,       // ← trophyId — came directly from TrophyQueue_PushUnlockRequest param_2
    &platflag)
```

The value `0x13` written into ring buffer word `[2]` by `TrophyQueue_PushUnlockRequest`
is read unchanged from linked list node word `[2]` by `TrophyManager_WorkerThread` and
passed verbatim as `trophyId` to `sceNpTrophyUnlockTrophy`. There is no transformation,
filtering, or re-mapping between push and unlock.

---

## 5. Full Lifecycle Diagram

```
   Any game thread                          TrophyManager_WorkerThread
   ─────────────                            ──────────────────────────
   sys_ppu_thread_create  ────────────────► _opd_FUN_011a7230
   (via FUN_008f9934)                              │ param_1 = mgr - 0xc
                                                   ▼
                                         sceNpTrophyRegisterContext
                                         sceNpTrophyGetTrophyUnlockState
                                         build unlock bitset @ +0x18
                                         sceNpTrophyGetTrophyInfo (per trophy)
                                         set ready flag @ +0x594b = 1
                                                   │
                                                   ▼  ◄─────────────────────┐
   Trophy_DispatchUnlockById(ctx, 0x13)          loop:                      │
     └► TrophyQueue_PushUnlockRequest            sys_mutex_lock(+0x34)      │
           write [0x13,...] → ctx+0x20           if queue empty:            │
     └► FUN_004b404c                               sys_cond_wait  ← sleeps  │
           move entry → ctx+0x50                 on signal: re-check ───────┘
           sys_cond_signal  ──────────────►      dequeue entry
                                                 trophyId = entry[2]
                                                 sys_mutex_unlock
                                                 sceNpTrophyUnlockTrophy(
                                                     ctx+0x10, ctx+0x14,
                                                     trophyId, &platflag)
                                                 update bitset @ +0x18
                                                 TrophyManager_OnTrophyUnlockedCallback
                                                 loop ──────────────────────┘
```

---

## 6. Key Addresses

| Symbol | Address | Notes |
|---|---|---|
| `TrophyManager_WorkerThread` | `0x004bdb60` | Worker thread body |
| `_opd_FUN_011a7230` | `0x011a7230` | OPD thunk; adjusts ptr by -0xc |
| `TrophyManager_OnTrophyUnlockedCallback` | `0x004bd968` | Post-unlock engine notification |
| `TrophyQueue_PushUnlockRequest` | `0x00cadc64` | Enqueues a trophy unlock event |
| `Trophy_DispatchUnlockById` | `0x00f2deb0` | Public dispatch helper |
| `FUN_004b404c` | `0x004b404c` | Transfers entry to worker FIFO + signals condvar |
| `FUN_008fa300` | `0x008fa300` | `sys_mutex_lock` wrapper |
| `FUN_008fa40c` | `0x008fa40c` | `sys_cond_wait` wrapper (lazy condvar creation) |
| `sys_ppu_thread_create` | `0x0144cf84` | PS3 LV2 import |
| OPD data pointer | `0x017ff1a0` | Points to `_opd_FUN_011a7230` entry |
| Thread entry fn-ptr slot | `0x017adfe0` | Data ref to OPD entry |
