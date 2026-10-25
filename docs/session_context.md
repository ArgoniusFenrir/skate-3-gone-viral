# Reverse Engineering Context & State: Skate 3 "Gone Viral" Trophy Analysis

**Date:** March 29, 2026 (Updated Session State)
**Target Binary:** Skate 3 (PS3) Title Update `v1.05` (`EBOOT.BIN` decrypted ELF)
**Tools:** Ghidra (PPC 64/32 Altivec `PowerISA-Altivec-64-32addr`), `Ps3GhidraScripts`, `GhidraMCP` (Bridge at `http://127.0.0.1:8080/`)
**Target Trophy:** Trophy 19 (`0x13`) — *Gone Viral* ("Catch the Skate Flu")

---

## 1. Key Function Reference & Memory Map

| Ghidra Symbol / Address | Type | Description / Annotations |
|---|---|---|
| `sceNpTrophyUnlockTrophy` (`0x0144cec4`) | PS3 SDK Import (`libnp.sprx`) | `int sceNpTrophyUnlockTrophy(SceNpTrophyContext context, SceNpTrophyHandle handle, SceNpTrophyId trophyId, SceNpTrophyFlag *platflag)`. Arg 3 (`r5`) is `trophyId` (19 / `0x13`). Export NID: `0x8CEEDD21`. |
| `TrophyManager_WorkerThread` (`0x004bdb60`) | Worker Thread | Central trophy background thread. Processes synchronized FIFO queue at `param_1 + 0x50`, calls `sceNpTrophyUnlockTrophy`, updates local bitmask `param_1 + 0x18`, calls `TrophyManager_OnTrophyUnlockedCallback`. |
| `TrophyManager_OnTrophyUnlockedCallback` (`0x004bd968`) | Callback Handler | Engine-level post-unlock notification dispatcher. |
| `Trophy_DispatchUnlockById` (`0x00f2deb0`) | Wrapper | Direct unlock dispatch helper. Calls `TrophyQueue_PushUnlockRequest(context, trophyId)`. |
| `TrophyQueue_PushUnlockRequest` (`0x00cadc64`) | Ring Queue Pusher | Enqueues trophy event `[trophyId, 0xffffffff, local_38, 0, 2]` onto active trophy ring queue (`param_1 + 0x20`). |
| `FUN_004b404c` (`0x004b404c`) | Queue Enqueue Helper | Enqueues unlock event requests into the internal worker thread queue (`param_1 + 0x50`). |
| `NetEvent_HandlePeerHandshake_GoneViralCheck` (`0x0045dc58`) | Network Event Handler | **Primary Viral Mechanism (FreeSkate/Lobby Join):** Receives event code `0x7b` (peer presence / handshake sync with infected player), triggers `Trophy_DispatchUnlockById(ctx, 0x13)`. |
| `OnlineMatch_OnEventProgressionComplete_TrophyDispatch` (`0x00448008`) | Match Progression Handler | **Secondary Viral Mechanism (Match Completion):** Evaluates player participant table post-match, triggers `FUN_004bd4ec` / `FUN_004bd57c` with trophy IDs. |
| `OnlineMatch_OnSessionEnd_TrophyDispatch` (`0x004488e8`) | Match Handler | Evaluates session end state for online progression trophy dispatch. |

---

## 2. Trigger Pathways & Complete Call Graph

```
Pathway 1: FreeSkate / Lobby Presence Handshake
───────────────────────────────────────────────
   FUN_0045dc58 (NetEvent_HandlePeerHandshake_GoneViralCheck / Event 0x7b)
        │ (Peer metadata flagged with Trophy 0x13)
        ▼
   FUN_00f2deb0 (Trophy_DispatchUnlockById, passes trophyId = 0x13)
        │
        ▼
   FUN_00cadc64 (TrophyQueue_PushUnlockRequest)
        │
        ▼
   FUN_004b404c (Enqueues unlock event into queue at param_1 + 0x50)
        │
        │
Pathway 2: Online Match Progression & Session End
─────────────────────────────────────────────────
   FUN_00448008 / FUN_004488e8 (OnlineMatch Progression & Session End Handlers)
        │ (Evaluates participant roster post-match)
        ▼
   FUN_004bd4ec / FUN_004bd57c (Trophy Event Dispatchers)
        │
        ▼
   FUN_004b404c (Enqueues unlock event into queue at param_1 + 0x50)
        │
        ├───────────────────────────────────────┘
        ▼
   FUN_004bdb60 (TrophyManager_WorkerThread: Background Worker Thread)
        │
        ├──► sceNpTrophyUnlockTrophy (0x0144cec4: Sony SDK System Import)
        │
        └──► FUN_004bd968 (TrophyManager_OnTrophyUnlockedCallback: Engine Notification)
```

---

## 3. API Signature & ABI Reference

- **Sony PS3 SDK (`libnp.sprx` / `np/trophy.h` / NID `0x8CEEDD21`):**
  - `int sceNpTrophyUnlockTrophy(SceNpTrophyContext context, SceNpTrophyHandle handle, SceNpTrophyId trophyId, SceNpTrophyFlag *platflag)`
  - PowerPC ABI registers: `r3` = context, `r4` = handle, `r5` = trophyId (`0x13` / 19), `r6` = platflag.
- **RPCS3 Reference:**
  - `error_code sceNpTrophyUnlockTrophy(ppu_thread& ppu, u32 context, u32 handle, s32 trophyId, vm::ptr<u32> platinumId)`
  - `ppu_thread& ppu` is emulator host harness context; guest registers correspond 1:1 (`r3`–`r6`).

---

## 4. Associated Project Files

- **Trophy Overview & Community Guides:** [`skate-3-gone-viral/docs/trophy_overview.md`](skate-3-gone-viral/docs/trophy_overview.md:1)
- **Binary & Code Analysis:** [`skate-3-gone-viral/docs/static_analysis.md`](skate-3-gone-viral/docs/static_analysis.md:1)
- **Investigation Plan:** [`skate-3-gone-viral/docs/binary_analysis_plan.md`](skate-3-gone-viral/docs/binary_analysis_plan.md:1)
- **Session Context:** [`skate-3-gone-viral/docs/session_context.md`](skate-3-gone-viral/docs/session_context.md:1)
- **MCP Config:** [`/home/ytbm/.bob/settings/mcp.json`](/home/ytbm/.bob/settings/mcp.json:1)
- **Ghidra Bridge Script:** [`ghidra/GhidraMCP-release-1-4/bridge_mcp_ghidra.py`](ghidra/GhidraMCP-release-1-4/bridge_mcp_ghidra.py:1)

---

## 5. Next Steps for Next Session

### Phase 1: Confirm Trigger Pathways via RPCS3 Debugger (Do First)

Before investigating the viral mechanism, we need to validate the two trigger pathways are actually functional using live RPCS3 debugging. This builds confidence in both the findings and the debugger workflow before deeper investigation.

#### Experiment 1 — Confirm the trophy worker thread exists

**Goal:** Verify at runtime that `FUN_004bdb60` (`TrophyManager_WorkerThread`) exists as a real, running thread in the emulated game, as the static analysis suggests.

**Steps:**
1. Launch Skate 3 (`v1.05` EBOOT) in RPCS3 with the debugger attached.
2. Once in-game, open the RPCS3 debugger's thread list and look for a PPU thread whose entry point is `0x004bdb60`.
3. If it does not show up in the thread list, set a breakpoint at `0x004bdb60` and restart the game from boot. The thread may be created early and then block on its queue.
4. Record the thread name, ID, and state (running / waiting).

**Expected outcome:** A PPU thread with entry point `0x004bdb60` exists and is idle, waiting on the FIFO queue at `param_1 + 0x50`.

**Notes:**
- Static analysis alone showed that this function drains the trophy queue. It has not yet been shown to run as its own thread.
- If no such thread exists, the trophy path in the call graph above needs to be re-examined before continuing.

#### Experiment 2 — Confirm `sceNpTrophyUnlockTrophy` is called for any trophy at runtime

**Goal:** Verify with the debugger that `sceNpTrophyUnlockTrophy` is actually called when the game awards a trophy. Using a trophy other than Gone Viral proves the call path works in general, before relying on it for Gone Viral.

**Test trophy:** "Do 3 manuals in a single line". It is easy to trigger anywhere.

**Steps:**
1. Launch Skate 3 (`v1.05` EBOOT) in RPCS3 with the debugger attached.
2. Set a breakpoint on the `sceNpTrophyUnlockTrophy` call inside `TrophyManager_WorkerThread`. The import is at `0x0144cec4` in Ghidra, so also try the import stub if the call site is not hit.
3. Perform 3 manuals in a single line.
4. When the breakpoint hits, record `r3` (context), `r4` (handle), and `r5` (`trophyId`).
5. Resume and confirm the trophy pops in-game and appears in the RPCS3 trophy list.

**Expected outcome:** The breakpoint is hit once, with `r5` holding the trophy ID for "Do 3 manuals in a single line". Record that ID, since it proves the call path and gives a known-good reference value.

**Notes:**
- If the breakpoint is never hit, check whether RPCS3 handles the import as a high-level emulated function (HLE) and move the breakpoint to the call site in the game code.
- The trophy may already be unlocked on the profile. Use a fresh profile or reset trophies in RPCS3 first.

#### Experiment 3 — Force-inject `trophyId = 0x13` into `TrophyManager_WorkerThread` via RPCS3 Debugger

**Goal:** Confirm we can manually drive Gone Viral unlock by patching the register state at the `TrophyManager_WorkerThread` entry point, without relying on the normal game trigger. Also builds confidence in modifying the instruction stream / register state in the RPCS3 debugger.

**Prerequisites:** Experiments 1 and 2 passed — the worker thread exists and `sceNpTrophyUnlockTrophy` is confirmed to be called for a regular trophy.

**Steps:**
1. Set a breakpoint at `0x004bdb60` (`TrophyManager_WorkerThread`).
2. Trigger any in-game event that causes *any* trophy to fire (same as Experiment 2) so the breakpoint is hit.
3. When execution pauses at `0x004bdb60`, open the RPCS3 register editor and **overwrite `r5` with `0x13`** (Gone Viral trophy ID).
4. Resume execution.
5. Observe whether Gone Viral (`trophyId = 0x13`) pops in-game despite the original trigger being for a different trophy.

**Expected outcome:** Gone Viral unlocks — confirms that the worker thread is the authoritative unlock gate and that `r5` at entry is the sole trophyId discriminator, and that register injection via RPCS3 debugger is viable for future experiments.

**Notes:**
- If the original trophy also pops (two trophies from one event), that is expected — the enqueued event for the original trophy was already committed to the FIFO before the breakpoint; the injection only affects this one dispatch call.
- If Gone Viral does not pop, examine whether `trophyId` is read from `r5` directly or from the queued struct at `param_1 + 0x50` — if the latter, the struct contents will need to be patched at memory level instead of just the register.
- Record the exact RPCS3 UI steps used (register panel location, edit flow) for documentation — this procedure will be reused in Experiments 4 and 5.

#### Experiment 4 — Confirm Pathway 1 (FreeSkate Handshake) hits `NetEvent_HandlePeerHandshake_GoneViralCheck`

- [ ] Set a breakpoint at `0x0045dc58` (`NetEvent_HandlePeerHandshake_GoneViralCheck`) and join an online FreeSkate session with an infected player; confirm event `0x7b` fires and the call chain reaches `TrophyManager_WorkerThread` with `r5 = 0x13`.

#### Experiment 5 — Confirm Pathway 2 (Match Completion) hits `OnlineMatch_OnEventProgressionComplete_TrophyDispatch`

- [ ] Set breakpoints at `0x00448008` and `0x004488e8`; complete an online match session and confirm the participant roster evaluation fires and reaches `TrophyManager_WorkerThread`.

---

### Phase 2: Network Mechanism Investigation (After Pathways Confirmed)

- [ ] Inspect the structure of network message `0x7b` / peer profile payload to see how the "infected" flag bit is packed in network packets.
- [ ] Investigate potential save-game flag or network emulation spoofing methods to simulate an infected peer.

---

### Phase 3: Closed-Loop Agentic AI Setup to Unlock Gone Viral via a Fake EA Blaze Server

**Goal:** Build a closed loop in which an AI agent starts the game, gets to a playable state, sends crafted network packets from a custom server that imitates EA's Blaze server, and checks whether Gone Viral unlocks. If it does not unlock, the agent changes the packet or goes back to Ghidra and tries again, until the trophy unlocks.

**Basis:** The findings in [`static_analysis.md`](static_analysis.md) show that event code `0x7b` in `NetEvent_HandlePeerHandshake_GoneViralCheck` leads directly to `Trophy_DispatchUnlockById(ctx, 0x13)`. The first packet to try is a TDF packet with command ID `0x7b`.

> **Caveat:** It is not yet known whether `0x7b` in the decompiled handler is the same value as a Blaze command ID on the wire. It may be an internal event code that the game derives from a different message. This is a hypothesis to test, not a confirmed fact. Phase 1 should help settle it.

#### Components

| Component | Role |
|-----------|------|
| **Orchestrator agent** | Runs the loop, keeps a log of every attempt (packet sent, result, reasoning), and decides the next step. |
| **Game launcher** | Starts RPCS3 with the Skate 3 `v1.05` boot path and restarts it between attempts. |
| **Input driver** | Sends controller input to RPCS3, for example through a virtual gamepad (Linux `uinput`) or RPCS3's keyboard-to-pad mapping. |
| **Screen / state observer** | Captures screenshots or RPCS3 log output so the agent can tell which menu the game is on. |
| **Fake Blaze server** | A custom server that imitates EA's Blaze server and answers the game's connection and login requests, then sends the crafted TDF packet. |
| **Network redirect** | Points the game's Blaze hostname at the fake server, using RPCS3's network settings, DNS override, or a hosts entry. |
| **Trophy checker** | Reads the emulator's user trophy file (`TROPUSR.DAT`, under the user's `trophy` folder in RPCS3's `dev_hdd0/home/`) and the RPCS3 log to see whether trophy `0x13` is unlocked. |
| **Ghidra via MCP** | Used by the agent for further static analysis when an attempt fails. |

#### The loop

1. **Reset:** Back up and reset the trophy state so Gone Viral is locked, and start from a known save.
2. **Start:** Launch RPCS3 with Skate 3 and the fake Blaze server running.
3. **Navigate:** Use controller input to press Start and X through the main menu, then load the latest save game. Confirm each screen with the observer before sending the next input.
4. **Connect:** Let the game connect to the fake Blaze server and complete login, answering whatever the game requires to stay online.
5. **Inject:** Send a TDF packet with command ID `0x7b`, with the payload fields chosen from the static analysis.
6. **Observe:** Wait a fixed time, then read the trophy file and RPCS3 log. Success means trophy `0x13` is now unlocked.
7. **Decide:**
   - **Unlocked:** Stop, save the exact packet and sequence that worked, and document them.
   - **Not unlocked:** Try a different packet (different TDF fields, command ID, component ID, or message sequence) and repeat from step 1.
   - **Still not unlocked after several variants:** Go back to Ghidra and continue static analysis of the packet parsing and the code that sets the infected state, update the findings, then build a new packet and repeat.
8. **Repeat** until Gone Viral unlocks.

#### Supporting steps

- [ ] Capture or reconstruct enough of the Blaze handshake to get the game online against the fake server (login, authentication, and any required component handshakes).
- [ ] Find the TDF encoding of the peer presence message by following the parsing code from `NetEvent_HandlePeerHandshake_GoneViralCheck` back to the network layer in Ghidra.
- [ ] Use the Phase 1 debugger breakpoints during injection attempts to see whether a packet reaches `0x0045dc58` even when the trophy does not unlock.
- [ ] Keep an attempt log (packet bytes, game state, breakpoint hits, trophy result) so the agent does not repeat failed attempts.
- [ ] Add a limit on attempts and a manual stop, so the loop cannot run forever.
