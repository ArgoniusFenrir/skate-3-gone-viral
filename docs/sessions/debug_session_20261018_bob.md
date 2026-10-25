# Debug Session 2026-10-18 — RPCS3 Patch Workflow Improvements

**Binary:** Skate 3 PS3 `v1.05` decrypted ELF
**Tool:** RPCS3 debugger + `imported_patch.yml` patch mechanism
**Goal:** Reduce overhead in the debug loop by patching out side-effects that prevented
re-triggering the same trophy in a single session.

---

## 1. Problem: The Existing Debug Loop Had Too Much Overhead

Before this session, testing the trophy unlock path required:

1. Set a breakpoint in the RPCS3 debugger
2. Boot the game
3. Trigger Mantastic Voyage (3+ manuals in a line)
4. Observe the breakpoint
5. **Restart the game entirely** before repeating — because the trophy was now marked as
   unlocked and the game would not attempt to unlock it again

The restart cost was significant. The goal for this session was to eliminate it by patching
the game's binary in memory via RPCS3's `imported_patch.yml` facility, which lets you
replace arbitrary instructions in the emulated address space at boot without modifying the
ELF on disk.

---

## 2. Patch 1 — Stub `sceNpTrophyUnlockTrophy` (Prevent Actual Trophy Commit)

### Motivation

The first patch was to stop the trophy from actually being committed to the RPCS3 profile.
This keeps the trophy in its locked state across sessions, so the same trigger can be
exercised repeatedly without resetting the profile.

### Patch

A NOP patch was applied at the `sceNpTrophyUnlockTrophy` call site inside
`TrophyManager_WorkerThread` (`0x004bdb60`), replacing the `bl` instruction with `nop` so
the SDK call never fires.

### Result

Confirmed via live debugger: the trophy no longer popped in-game and did not appear in the
RPCS3 trophy list after triggering Mantastic Voyage. The patch worked as intended.

---

## 3. Problem Exposed by Patch 1 — Triggering the Same Trophy a Second Time Did Nothing

After Patch 1, triggering Mantastic Voyage a second time in the same session produced no
activity. The trophy worker thread was never woken up — no breakpoint on
`_opd_FUN_004acbc8` or `TrophyManager_WorkerThread` was hit on the second attempt.

### Root Cause

Prior debug notes and AI-inferred pseudocode in the repo mentioned that after
`sceNpTrophyUnlockTrophy` is called, a **local bitset vector** (later confirmed to live at
`ctx + 0x18` on the trophy manager object) is updated to record the trophy as unlocked.

This bitset is checked **before** enqueueing a trophy request — it acts as an early-exit
gate in `FUN_004b11d4` / `FUN_004b12ec` to avoid re-queuing a trophy that has already
been awarded. Because the NOP patch prevented `sceNpTrophyUnlockTrophy` from running but
did **not** prevent the bitset update, the game still recorded Mantastic Voyage as unlocked
in the bitset after the first trigger, and silently dropped every subsequent request.

---

## 4. Patch 2 — Disable the Local Bitset Update (Allow Re-triggering)

### Patch

A second patch was applied to the instruction(s) inside `TrophyManager_WorkerThread` that
write the trophy ID into the local bitset at `ctx + 0x18`, replacing them with `nop`s so
the "already unlocked" flag is never set.

### Result

With both patches applied:

- Triggering Mantastic Voyage a second time (and subsequent times) in the same session
  **worked as expected**.
- The trophy worker thread was woken each time.
- Breakpoints on the worker thread path fired on every trigger, not just the first.
- The trophy never popped in the RPCS3 profile (Patch 1 still active).

This gave a clean, repeatable debug loop without restarting the game.

---

## 5. Patches Applied to `imported_patch.yml`

Both patches were committed to the repository under the project's patch file.

| Patch | Address | Change | Purpose |
|---|---|---|---|
| NOP `sceNpTrophyUnlockTrophy` call | Inside `TrophyManager_WorkerThread` | `bl → nop` | Prevents trophy committing to RPCS3 profile |
| NOP bitset update | Inside `TrophyManager_WorkerThread` | `stb/stw → nop` | Prevents "already unlocked" flag being set, allows re-triggering |

---

## 6. Key Insight

The game has **two separate "already unlocked" stores** that work independently:

| Store | Location | Set by | Checked by |
|---|---|---|---|
| Local in-process bitset | `trophy_manager_ctx + 0x18` | `TrophyManager_WorkerThread` after `sceNpTrophyUnlockTrophy` | `FUN_004b12ec` — called in `FUN_004b11d4` before every enqueue |
| PS3/RPCS3 system trophy state | RPCS3 `TROPUSR.DAT` / `dev_hdd0` | `sceNpTrophyUnlockTrophy` SDK call | SDK itself — returns error if already unlocked |

Stubbing only the SDK call left the local bitset intact, which silently blocked
re-triggering. Both had to be disabled to get a fully repeatable loop.
