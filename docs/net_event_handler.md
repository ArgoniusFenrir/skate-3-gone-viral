# NetEvent_HandlePeerHandshake_GoneViralCheck — Event Dispatch Analysis

**Binary:** Skate 3 PS3 `v1.05` decrypted ELF  
**Function:** `NetEvent_HandlePeerHandshake_GoneViralCheck` @ `0x0045dc58`  
**Analysis method:** Ghidra decompilation via GhidraMCP (live session)

---

## 1. Function Signature

```c
undefined8 NetEvent_HandlePeerHandshake_GoneViralCheck(int param_1, int *param_2)
```

| Parameter | Type | Meaning |
|---|---|---|
| `param_1` | `int` | Pointer to the local player/session context object |
| `param_2` | `int *` | **Pointer to a pointer** to the incoming network event object |

---

## 2. Event Object Extraction — The Double Dereference

The first two lines of the function extract the event type ID:

```c
iVar12 = *param_2;           // deref param_2 → address of the event object
iVar7  = *(int *)(iVar12 + 8); // read int at offset +8 → event type ID
```

### Type chain

```
param_2          int *     pointer to an event object pointer (EventObject **)
*param_2         int       address of the event object on the heap / ring buffer (iVar12)
*(iVar12 + 8)    int       32-bit event type ID field at byte offset +8 (iVar7)
```

`param_2` is a **double-indirection pointer** — a common event-bus ABI in C++ game
engines. The dispatcher holds a slot (`param_2`) that points to the current event
object on the heap or in a ring buffer. The handler dereferences the slot to get the
object, then reads the type field to decide what to do.

---

## 3. Inferred Event Object Layout

All field accesses observed across the handler:

| Byte offset | Access width | Used for |
|---|---|---|
| `+0x00` | — | object base / identity |
| `+0x08` | `int` (32-bit) | **Event type ID** — primary dispatch key (`iVar7`) |
| `+0x0c` | `char` (8-bit) | Boolean flag (checked under type `0x2c`) |
| `+0x10` | block start | Peer/player data block (passed as pointer to sub-functions) |
| `+0x50` | `uint` (32-bit) | Peer data field |
| `+0x54` | `uint8` | Peer data field |
| `+0x55` | `uint8` | Peer data field |
| `+0x56` | `uint8` | Peer data field |
| `+0x8c` | `uint8` | Peer data field |

---

## 4. Event Type Dispatch Table

`iVar7` (the event type ID at offset `+8`) is used as a manual dispatch switch
implemented as a flat `if / else if` chain. Ghidra did not produce a jump table
because the type values are sparse.

| `iVar7` value | Symbolic name (inferred) | Action |
|---|---|---|
| `0x02` | Trick replay | Reads trick type, calls `FUN_00446968` to apply trick state |
| `0x03` | Player join / slot assignment | Assigns player to slot, calls `FUN_0041ec58` |
| `0x04` | Bail | Sets bail flag, calls `FUN_0041ec58` |
| `0x2c` | Session flag update | Checks flag at `iVar12 + 0xc`, calls `FUN_0041ec58` if set |
| `0x7b` | **Peer handshake / lobby presence** | **Calls `Trophy_DispatchUnlockById(ctx, 0x13)` → Gone Viral unlock** |
| `0x7c` | Trick type sync | Calls `FUN_00f2e510`; sets `param_1 + 0xd8 = 2` |
| `0x7d` | State update | Sets `param_1 + 0xd8 = 3` |
| anything else | Unhandled | Returns `0` |

---

## 5. Gone Viral Trigger — Event `0x7b` Code Path

```c
if (iVar7 != 0x7b) {
    return 0;
}

FUN_0041ec58(param_1, 0, 0, 1);          // session state update

iVar12 = *(int *)PTR_s_Scale_0182bbf0;   // load trophy dispatch context pointer

if (iVar12 == 0) {
    // Lazy-initialize the trophy dispatch context
    uVar9 = *(undefined4 *)PTR_PTR_0182c200;
    uVar6 = FUN_006ab308(PTR_s_IsOnBoard_0182bc60);
    uVar9 = FUN_006ab548(0xa8, uVar6, uVar9, 0x101, 8, 0);
    *(undefined4 *)PTR_s_Scale_0182bbf0 = uVar9;
    FUN_00f2cc80(uVar9);
    uVar6 = FUN_008fa6dc();
    iVar12 = *(int *)PTR_s_Scale_0182bbf0;
    **(undefined8 **)(iStack_c4 + -0x6b64) = uVar6;
}

Trophy_DispatchUnlockById(iVar12, 0x13); // enqueue Gone Viral unlock
```

Key observations:

- There is **no condition check on peer state** before calling
  `Trophy_DispatchUnlockById`. Receiving event `0x7b` is sufficient on its own.
- A trophy dispatch context is stored at the global pointer
  `PTR_s_Scale_0182bbfO`. If it is null (first call), it is lazily constructed
  via `FUN_006ab548` before the dispatch call.
- `Trophy_DispatchUnlockById(ctx, 0x13)` feeds directly into
  `TrophyQueue_PushUnlockRequest` → worker FIFO → `sceNpTrophyUnlockTrophy`.
  See [`trophy_manager_thread.md`](trophy_manager_thread.md) for that path.

---

## 6. Call Site — How the Handler Is Invoked

`NetEvent_HandlePeerHandshake_GoneViralCheck` is **called from exactly one place**
in the binary. There is no direct `bl` (branch-and-link) call to `0x0045dc58`
anywhere. The only xref to its address is a single DATA reference: its OPD entry
(`0x017fc270`) is stored as slot `+0x14` in a **state machine handler dispatch
table** based at `0x017aaf40`.

### The dispatch table (`0x017aaf40` – `0x017aaf60`)

This is a 9-entry table of pointers to OPD entries (4 bytes each). Each entry is a
function pointer to one handler in the same family. The entry for
`NetEvent_HandlePeerHandshake_GoneViralCheck` is the fifth slot (`+0x14`).

| Table offset | Slot address | OPD ptr | Function | Role |
|---|---|---|---|---|
| `+0x00` | `0x017aaf40` | `0x017fb3f0` | `.opd.FUN_0043372c` | Event `0x37` handler |
| `+0x04` | `0x017aaf44` | `0x017fb440` | `.opd.FUN_004337d0` | Destructor |
| `+0x08` | `0x017aaf48` | `0x017fb438` | `.opd.FUN_0043385c` | Destructor + delete |
| `+0x0c` | `0x017aaf4c` | `0x017f99e0` | `.opd.FUN_011678e4` | (unknown) |
| `+0x10` | `0x017aaf50` | `0x017fa310` | `.opd.FUN_0041fbcc` | Null stub (returns 0) |
| **`+0x14`** | **`0x017aaf54`** | **`0x017fc270`** | **`NetEvent_HandlePeerHandshake_GoneViralCheck`** | **Gone Viral trigger** |
| `+0x18` | `0x017aaf58` | `0x017fb538` | `.opd.FUN_00433900` | Session flag handler |
| `+0x1c` | `0x017aaf5c` | `0x017faec0` | `.opd.FUN_00427b98` | Trick type sync |
| `+0x20` | `0x017aaf60` | `0x017fa318` | `.opd.FUN_0041fbd4` | State init / reset |

### How the dispatch works

The table belongs to a **network-connected player state machine** object. The engine
framework loads the table pointer from the object's vtable slot, indexes into it at
`+0x14`, and calls through it, passing `(stateObject, eventPtr)`. This is a
two-level indirection:

```
stateObj->vtable[n]  →  handler dispatch table base (0x017aaf40)
table[+0x14]         →  OPD ptr (0x017fc270)
OPD ptr              →  NetEvent_HandlePeerHandshake_GoneViralCheck (0x0045dc58)
```

The state machine object is constructed by `FUN_0043899c` (via factory
`_opd_FUN_01170b4c` which allocates `0x5c` bytes). Its base constructor
`FUN_0041e620` writes the base vtable (`PTR_PTR_0182b7f8`); `FUN_0043899c`
overwrites it with the concrete vtable (`PTR_PTR_0182c070`). The dispatch table
at `0x017aaf40` is a separate handler table referenced within that vtable
hierarchy, accessed at runtime via TOC-relative addressing (no direct code xref
in Ghidra).

### Summary

There is **one and only one call site** for `NetEvent_HandlePeerHandshake_GoneViralCheck`:
the state machine dispatcher that indexes the handler table at `0x017aaf40 + 0x14`
and calls through it. Any path that reaches the Gone Viral unlock must go through
this single dispatch point.

---

## 7. Implications for Fake Blaze Server

Event type `0x7b` is the **only value in this handler that unlocks Gone Viral**, and
it does so unconditionally on receipt. The open question (noted in
[`session_context.md`](session_context.md)) is whether `0x7b` at offset `+8` of the
event object is the same as a Blaze wire command/component ID, or whether the game
maps Blaze message fields onto this internal event type upstream of this handler.

To answer that, the next step is to trace what constructs and dispatches the event
object upstream — i.e. find the function that allocates an event object, writes
`0x7b` to offset `+8`, and enqueues it into the state machine bus that eventually
calls through `0x017aaf40 + 0x14`.

---

## 8. Key Addresses

| Symbol | Address | Notes |
|---|---|---|
| `NetEvent_HandlePeerHandshake_GoneViralCheck` | `0x0045dc58` | This function |
| `Trophy_DispatchUnlockById` | `0x00f2deb0` | Called with `trophyId = 0x13` on event `0x7b` |
| `FUN_0041ec58` | `0x0041ec58` | Session state updater; called for several event types |
| `FUN_00446968` | `0x00446968` | Peer slot / trick state applicator (types `0x02`, `0x03`) |
| `PTR_s_Scale_0182bbf0` | `0x0182bbf0` | Global pointer to trophy dispatch context |
| Handler dispatch table | `0x017aaf40` | 9-entry table; `NetEvent` handler is at `+0x14` |
| OPD entry for `NetEvent` handler | `0x017fc270` | Only xref to `0x0045dc58` |
| State machine base constructor | `0x0041e620` | Writes base vtable `PTR_PTR_0182b7f8` |
| State machine concrete constructor | `0x0043899c` | Overwrites vtable with `PTR_PTR_0182c070` |
| State machine factory | `0x01170b4c` | Allocates `0x5c`-byte object, calls `FUN_0043899c` |
