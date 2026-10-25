# Bob Session Log — Skate 3 Gone Viral Trophy RE

Chronological summary of every session worked on together.
One entry per distinct working day. Dates marked **~estimated** where the source
material contains no explicit date.

---

## ~Early October 2026 (estimated) — Project Setup & Trophy Research

**Sources:** `trophy_overview.md`, `README.md`, `binary_analysis_plan.md`

### What we did

- Established the project goal: reverse-engineer the *Gone Viral* trophy (ID `0x13` / 19)
  in Skate 3 PS3 `v1.05` to understand exactly how it unlocks at runtime, with the
  long-term goal of building a fake EA Blaze server that can trigger it programmatically.
- Researched the trophy from community sources (PSNProfiles, PlayStationTrophies.org,
  TrueAchievements). All confirmed the same behaviour: it spreads virally — playing online
  with someone who already holds it passes it on automatically.
- Parsed the game's `TROPHY.TRP` archive. Found the canonical entry:
  ```xml
  <trophy id="019" hidden="yes" ttype="B" pid="000">
    <name>Gone Viral</name>
    <detail>Catch the Skate Flu</detail>
  </trophy>
  ```
  Confirmed decimal ID `019` = hex `0x13` = the value that must be passed to
  `sceNpTrophyUnlockTrophy` as `trophyId`.
- Prepared the analysis environment:
  - Obtained the v1.05 update `.pkg` via `rusty-psn`, unpacked and decrypted
    `EBOOT.BIN` to a plain ELF using RPCS3's built-in decryption.
  - Imported the ELF into Ghidra targeting `PowerISA-Altivec-64-32addr` (big-endian
    PPC 32-bit address space).
  - Applied `Ps3GhidraScripts` and updated `ppc_64_32.cspec` per the script's README.
  - Configured `GhidraMCP` to bridge Ghidra's CodeBrowser to an HTTP API at
    `http://127.0.0.1:8080/` for agentic AI-driven decompilation.
- Wrote `binary_analysis_plan.md` as a structured sub-task list to drive the analysis.

---

## ~Early–Mid October 2026 (estimated) — Static Analysis Phase 1: Call Graph

**Sources:** `static_analysis.md`, `binary_analysis_plan.md`

### What we did

- Drove Ghidra via GhidraMCP (agentic AI) to cross-reference `sceNpTrophyUnlockTrophy`
  (NID `0x8CEEDD21`, import at `0x0144cec4`) across the full binary.
- Confirmed the PS3 SDK signature and RPCS3 open-source implementation ABI:
  - `r3` = NP context, `r4` = request handle, `r5` = `trophyId`, `r6` = platflag pointer.
- Found a single central **trophy worker thread** (`FUN_004bdb60`) that:
  - Drains a synchronized FIFO queue at `param_1 + 0x50`
  - Reads `trophyId` from `entry[2]` of each dequeued node
  - Calls `sceNpTrophyUnlockTrophy` and then updates a local 128-bit bitset at
    `param_1 + 0x18`
- Traced the enqueue path back to two named helpers:
  - `FUN_00cadc64` → named `TrophyQueue_PushUnlockRequest` — writes a 5-word entry
    `[trophyId, 0xffffffff, trophyId, 0, 2]` into a ring buffer at `ctx + 0x20`
  - `FUN_00f2deb0` → named `Trophy_DispatchUnlockById` — thin wrapper that calls
    `TrophyQueue_PushUnlockRequest`
  - `FUN_004b404c` — transfers entries from the ring buffer into the worker FIFO and
    calls `sys_cond_signal` to wake the worker
- Found **two trigger pathways** for Gone Viral (`0x13`):
  - **Pathway 1 (FreeSkate / Lobby Join):** `FUN_0045dc58` handles network event `0x7b`
    and calls `Trophy_DispatchUnlockById(ctx, 0x13)` unconditionally on receipt.
  - **Pathway 2 (Match/Session End):** `FUN_00448008` / `FUN_004488e8` evaluate the
    online participant roster post-match and dispatch through `FUN_004bd4ec` /
    `FUN_004bd57c` into `FUN_004b404c`.
- Named three functions based on static inference (later partially reverted):
  - `Trophy_DispatchUnlockById` (`0x00f2deb0`)
  - `TrophyQueue_PushUnlockRequest` (`0x00cadc64`)
  - `TrophyManager_OnTrophyUnlockedCallback` (`0x004bd968`)
- Documented all findings in `static_analysis.md`.

> **Status at end of this phase:** full static call graph mapped and documented.
> No runtime confirmation yet — all function names and inferences are from
> decompilation only.

---

## ~Mid October 2026 (estimated) — Static Analysis Phase 2: Deep Dives

**Sources:** `trophy_manager_thread.md`, `net_event_handler.md`

### What we did

These were two focused static analysis sessions that drilled into specific parts of
the call graph to produce detailed documentation.

#### Trophy Manager Worker Thread (`trophy_manager_thread.md`)

- Decompiled `TrophyManager_WorkerThread` (`0x004bdb60`) in full.
- Documented the thread creation call chain:
  `FUN_008f9934` → `sys_ppu_thread_create` → OPD thunk `_opd_FUN_011a7230` (adjusts
  `this` pointer by `-0xc`) → `TrophyManager_WorkerThread`.
- Documented the startup sequence: `sceNpTrophyRegisterContext` →
  `sceNpTrophyGetTrophyUnlockState` → build local unlock bitset at `+0x18` →
  `sceNpTrophyGetGameInfo` → per-trophy `sceNpTrophyGetTrophyInfo` loop → set ready flag
  at `+0x594b`.
- Documented the main dispatch loop: mutex-lock → check queue → if empty: `sys_cond_wait`
  (blocks) → on wake: dequeue node → read `trophyId = node[2]` → call
  `sceNpTrophyUnlockTrophy` → update bitset → notify callback → loop.
- Proved field-by-field that the trophy ID written by `TrophyQueue_PushUnlockRequest`
  into ring buffer word `[2]` survives intact through `FUN_004b404c`'s list insertion
  (words `[0]` and `[1]` are overwritten by list link pointers; word `[2]` is not
  touched) and is read unchanged by the worker as `uVar16` → passed as `r5` to
  `sceNpTrophyUnlockTrophy`.

#### Network Event Handler (`net_event_handler.md`)

- Decompiled `FUN_0045dc58` (named `NetEvent_HandlePeerHandshake_GoneViralCheck`) in full.
- Mapped the event object layout — type ID at byte offset `+0x8`, accessed via double
  indirection through `param_2`.
- Reconstructed the full `if/else if` event dispatch chain and documented all 7 event
  type branches (`0x02`, `0x03`, `0x04`, `0x2c`, `0x7b`, `0x7c`, `0x7d`).
- Confirmed event `0x7b` calls `Trophy_DispatchUnlockById(ctx, 0x13)` with **no
  condition check on peer state** — receiving the event is sufficient.
- Found that the function has **no direct `bl` call site** anywhere in the binary —
  it is invoked exclusively through a 9-entry handler dispatch table at `0x017aaf40`,
  at slot `+0x14`, via two levels of vtable indirection from a network-connected player
  state machine object.
- Noted the open question: whether internal event code `0x7b` at offset `+8` of the
  event object corresponds to a Blaze wire command/component ID, or whether the game
  maps Blaze fields onto this value upstream.
