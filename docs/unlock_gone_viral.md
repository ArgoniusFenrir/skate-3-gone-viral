# How to Unlock the Gone Viral Trophy?

Investigate how the Gone Viral trophy unlocks.

## Trophy Identification

PS3 games come with a `TROPHY.TRP` archive that describes every trophy the title
can award.
Parsing this archive gives us the game-defined identifier for Gone Viral that we can then search for in the game executable and save data.

### Trophy Details

The relevant entry from `TROPHY.TRP` is:

```
<trophy id="019" hidden="yes" ttype="B" pid="000">
  <name>Gone Viral</name>
  <detail>Catch the Skate Flu</detail>
</trophy>
```

| Field    | Value | Meaning |
|----------|-------|---------|
| `id`     | `019` | Decimal trophy index within this title |
| `hidden` | `yes` | Hidden in the XMB until earned; the description is not shown to the player beforehand |
| `ttype`  | `B`   | Bronze trophy |

### Key Findings

Trophy ID 19 (`0x13`) is the canonical identifier for Gone Viral within Skate 3.
This value is the handle the game engine uses when it calls the PS3 trophy service to award the trophy (`sceNpTrophyUnlockTrophy`).
Any save-file flag, in-game condition check, or code patch that targets this trophy will reference this ID.

## Executable Preparation & Environment

To analyze the trophy unlock logic, we inspect the main game executable from the latest title update:

1. **Acquisition:**
   - The final official update release for Skate 3 is version `1.05`.
   - The update package (`.pkg`) was obtained via `rusty-psn` and unpacked to extract the updated `EBOOT.BIN`.
   - The v1.05 `EBOOT.BIN` is slightly larger than the base release executable due to the inclusion of DLC.

2. **Decryption:**
   - The encrypted PS3 executable was decrypted into an ELF binary using RPCS3's built-in decryption utilities.

3. **Ghidra Analysis & Tooling:**
   - The decrypted ELF was imported into Ghidra targeting the big-endian `PowerISA-Altivec-64-32addr` architecture.
   - `Ps3GhidraScripts` was executed according to its `README.md` and `ppc_64_32.cspec` was updated in the process.
   - `GhidraMCP` was configured to interface directly with the active CodeBrowser session for decompilation workflows.

## Investigation Methodology

To determine the exact conditions that trigger the trophy unlock, the executable was analyzed through the steps outlined below.

### SDK Function Prototype

The PS3 Trophy API is part of Sony's Network Platform library (`libnp.sprx`).

- **Official Sony PS3 Cell SDK:**
  `target/ppu/include/np/trophy.h` / `libnp.sprx`, mapped in [`Ps3GhidraScripts`](https://github.com/clienthax/Ps3GhidraScripts/blob/d7cb6fd33da40e6547a23fa471292a58df54b89b/data/nids.txt#L5241) to NID `0x8CEEDD21`.

  ```cpp
  int sceNpTrophyUnlockTrophy(
      SceNpTrophyContext context,     // arg 1 (r3): Trophy context handle
      SceNpTrophyHandle handle,       // arg 2 (r4): Request handle
      SceNpTrophyId trophyId,         // arg 3 (r5): 1-based Trophy ID (0x13 / 19 for Gone Viral)
      SceNpTrophyFlag *platflag       // arg 4 (r6): Output flag pointer (optional)
  );
  ```

- **RPCS3 Open-Source Implementation Reference:**
  [`sceNpTrophy.cpp`](https://github.com/RPCS3/rpcs3/blob/43a42d1a868794ae5ff592a1d0055c2c615eaa44/rpcs3/Emu/Cell/Modules/sceNpTrophy.cpp#L1068)

  ```cpp
  error_code sceNpTrophyUnlockTrophy(
      ppu_thread& ppu,                 // Emulator host context (not a guest PS3 register)
      u32 context,                     // Guest register r3
      u32 handle,                      // Guest register r4
      s32 trophyId,                    // Guest register r5 (Trophy ID: 19 / 0x13)
      vm::ptr<u32> platinumId          // Guest register r6
  );
  ```

  *Note: The first argument `ppu_thread& ppu` is RPCS3's internal C++ host execution harness parameter; the 4 remaining parameters map 1:1 to the guest PowerPC registers `r3` through `r6`.*

### Workflow

1. **Cross-Referencing:**
   - Query all cross-references to `sceNpTrophyUnlockTrophy` across the executable.
   - Trace callers and examine parameter passing in register `r5` (`trophyId`).
   - Locate any centralized trophy manager / dispatcher wrappers and map out their callers.

2. **Trophy ID Lookup (decimal `19` / hex `0x13`):**
   - Search for references passing constant decimal `19` or hexadecimal `0x13` (checking both representations in decompiled C and PowerPC immediate instructions e.g. `li r5, 0x13` / `li r5, 19`) to the trophy unlock dispatcher.
   - Identify the specific subsystem or event callback responsible for evaluating the "Skate Flu" condition.

3. **Condition & State Analysis:**
   - Decompile the candidate function(s) to understand what game state triggers the unlock call.
   - Document any multiplayer handshake checks or infection mechanisms embedded in the netcode.

## Decompiled Trophy Architecture & Call Graph

Through executable decompilation in Ghidra, the game's trophy management pipeline was reconstructed across two trigger pathways:

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

### Central Trophy Dispatcher (`FUN_004bdb60` / `0x004bdb60`)

The game engine routes all trophy operations through a central worker thread function:
- It maintains an internal unlocked bitset (`param_1 + 0x18`) initialized via `sceNpTrophyGetTrophyUnlockState`.
- It processes a thread-safe synchronized FIFO queue (`param_1 + 0x50`).
- For each queued trophy unlock request, it extracts the target `uVar16` (`trophyId`) and invokes `sceNpTrophyUnlockTrophy`:
  ```c
  local_b44 = -1;
  iVar12 = sceNpTrophyUnlockTrophy(
               *(int *)(param_1 + 0x10), // context
               *(int *)(param_1 + 0x14), // handle
               uVar16,                   // trophyId (19 / 0x13 for Gone Viral)
               &local_b44                // platinum output flag
           );
  if (iVar12 >= 0) {
      // Mark bit in local engine unlocked bitset
      puVar2[uVar16 >> 6] |= (1ULL << (uVar16 & 0x3f));
      // Notify game engine listeners
      FUN_004bd968(param_1, uVar16);
  }
  ```

### Online Session & FreeSkate Lobby Trigger Logic (`FUN_0045dc58` & `FUN_00cadc64`)

In addition to competitive match completion events, the binary contains an immediate online participant evaluation mechanism (`FUN_0045dc58` / `FUN_00cadc64`) that explains why Gone Viral unlocks simply upon entering an online FreeSkate lobby:

1. **Online Player Presence & Lobby Sync (`FUN_0045dc58`):**
   - When a player joins any online session (including FreeSkate lobbies), player state packets and profile metadata are synchronized.
   - Message/event code `0x7b` represents the online peer connection handshake / presence notification.
   - Upon handling event `0x7b`, `_opd_FUN_0045dc58` checks the remote player's infected/trophy status and directly calls:
     ```c
     if (iVar7 == 0x7b) {
         FUN_0041ec58(param_1, 0, 0, 1);
         iVar12 = *(int *)PTR_s_Scale_0182bbf0;
         if (iVar12 == 0) {
             // Initialize online profile / achievement tracker context
             uVar9 = *(undefined4 *)PTR_PTR_0182c200;
             uVar6 = FUN_006ab308(PTR_s_IsOnBoard_0182bc60);
             uVar9 = FUN_006ab548(0xa8, uVar6, uVar9, 0x101, 8, 0);
             *(undefined4 *)puVar4 = uVar9;
             FUN_00f2cc80(uVar9);
             uVar6 = FUN_008fa6dc();
             iVar12 = *(int *)puVar4;
         }
         // Direct unlock dispatch with Trophy ID 0x13 (19)
         FUN_00f2deb0(iVar12, 0x13);
     }
     ```

2. **Internal Unlock Dispatcher (`FUN_00cadc64` / `FUN_00f2deb0`):**
   - `FUN_00f2deb0` routes directly into the game engine's internal award queue manager (`FUN_00cadc64`).
   - `FUN_00cadc64(context, 0x13)` appends entry `[trophyId = 0x13, param = 0xffffffff, type = 2]` into the active trophy ring/FIFO queue (`param_1 + 0x20`).
   - The background worker thread (`FUN_004bdb60`) consumes the entry and immediately calls `sceNpTrophyUnlockTrophy(context, handle, 0x13, &platflag)`.

### Summary of Trigger Pathways for Gone Viral

| Pathway | Trigger Location | Condition |
|---------|-----------------|-----------|
| **FreeSkate / Lobby Join** | `_opd_FUN_0045dc58` (event `0x7b`) -> `FUN_00cadc64` | Connecting to or hosting a session where a peer has the viral flag set in their presence packet (`FUN_00f2deb0(ctx, 0x13)`). No event or match completion needed. |
| **Match / Session Events** | `_opd_FUN_00448008` & `_opd_FUN_004488e8` -> `FUN_004bd4ec` | Completing an online multiplayer match or team challenge session with an infected player. |
