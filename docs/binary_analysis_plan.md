# Plan: Binary Analysis of Trophy 19 (Gone Viral) via Ghidra MCP

## Overview
Reverse-engineer the Skate 3 (v1.05) executable using the connected Ghidra MCP server to determine the exact conditions, network/session flags, and mechanics that trigger Trophy 19 (`Gone Viral` / "Catch the Skate Flu"). Once analyzed, document the decompiled findings into `skate-3-gone-viral/docs/static_analysis.md`.

---

## Sub-Tasks

### Sub-Task 1: Find and Analyze All `sceNpTrophyUnlockTrophy` Call Sites
- **Intent**: Query all cross-references to the PS3 SDK `sceNpTrophyUnlockTrophy` import and analyze every call site in the binary using the known Sony PS3 SDK signature.
- **Expected Outcomes**: A complete inventory of every location calling `sceNpTrophyUnlockTrophy` (e.g. internal wrapper functions, direct subsystem calls), tracking argument passing (specifically argument 3: `trophyId`), and mapping out all upstream callers.
- **Todo List**:
  - [x] Query all references (code & data XREFs) to `sceNpTrophyUnlockTrophy`.
  - [x] Set/verify the function prototype for `sceNpTrophyUnlockTrophy` in Ghidra (`int sceNpTrophyUnlockTrophy(SceNpTrophyContext context, SceNpTrophyHandle handle, SceNpTrophyId trophyId, SceNpTrophyFlag *platflag)`).
  - [x] Decompile and analyze each call site / referencing function to determine whether it is a central wrapper, dispatcher, or direct unlock invocation.
  - [x] For any identified wrapper/dispatcher functions, track which parameter receives the `trophyId` (arg 3 / `r5` in PPC ABI) and query their upstream XREFs across the binary.
  - [x] Filter all discovered paths for where Trophy ID 19 (`0x13`) is passed into `trophyId` or checked.
- **Relevant Context**:
  - `sceNpTrophyUnlockTrophy` signature: `int sceNpTrophyUnlockTrophy(SceNpTrophyContext context, SceNpTrophyHandle handle, SceNpTrophyId trophyId, SceNpTrophyFlag *platflag)`
  - Sources & References:
    - Official PlayStation 3 SDK: `cell/sysmodule/np/np_trophy.h` (`libnp.sprx` / `libsysmodule.sprx`)
    - PS3 Developer Wiki: [PS3 Dev Wiki - sceNpTrophy](https://www.psdevwiki.com/ps3/Np_trophy) / [libnp](https://www.psdevwiki.com/ps3/Libnp)
    - Open-source implementations & headers (RPCS3 emulator source: `rpcs3/Emu/Cell/Modules/sceNpTrophy.cpp` & `sceNpTrophy.h`, PSL1GHT SDK: `np/trophy.h`)
  - PowerPC 64/32 ABI calling convention: arg 1 = `r3` (context), arg 2 = `r4` (handle), arg 3 = `r5` (`trophyId`), arg 4 = `r6` (platflag)
  - Ghidra MCP XREF, prototype, and decompiler tools
- **Status**: `[x] done`

---

### Sub-Task 2: Decompile & Analyze the "Gone Viral" Unlock Handler
- **Intent**: Analyze the specific function(s) that issue the unlock call for Trophy 19.
- **Expected Outcomes**: Determine the exact trigger conditions, multiplayer session checks, player attribute comparisons, or infection packet handlers.
- **Todo List**:
  - [x] Search and filter candidate call sites and constant assignments for both decimal `19` and hexadecimal `0x13` values.
  - [x] Decompile the target function(s) passing `19` / `0x13` to the unlock dispatcher.
  - [x] Trace referenced data structures (player state, session player list, online packet fields, save profile flags).
  - [x] Check for string references related to "flu", "viral", or online multiplayer events.
  - [x] Identify whether the trigger is evaluated on player join, match end, contact/proximity, or teammate status.
- **Relevant Context**:
  - Trophy ID: decimal `19` / hex `0x13` (look for both representations in decompilation and immediate PPC assembly operands like `li rX, 0x13` or `li rX, 19`)
  - Decompiler & XREF MCP tools
- **Status**: `[x] done`

---

### Sub-Task 3: Document Findings in `static_analysis.md`
- **Intent**: Summarize the technical mechanics, pseudocode, and unlock conditions in the documentation.
- **Expected Outcomes**: Updated `skate-3-gone-viral/docs/static_analysis.md` containing the full reverse engineering results under the Binary Analysis section.
- **Todo List**:
  - [ ] Format decompiled logic and internal functions into readable documentation.
  - [ ] Detail the exact viral transmission logic (how player A with the trophy infects player B).
  - [ ] Note potential offline/emulation bypasses or unlocking triggers.
- **Relevant Context**:
  - [`skate-3-gone-viral/docs/static_analysis.md`](skate-3-gone-viral/docs/static_analysis.md:1)
- **Status**: `[x] done`
