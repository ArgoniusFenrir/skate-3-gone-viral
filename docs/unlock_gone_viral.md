# Unlock Gone Viral

Investigate how the Gone Viral trophy unlocks.

## Trophy Identification

### Background

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
