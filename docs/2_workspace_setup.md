# Workspace Setup

Workspace setup for the reverse engineering efforts.

## Tools

The following tools were used to decompress binaries, analyze code, and verify findings.

| Tool | Purpose |
|------|---------|
| [Ghidra](https://github.com/NationalSecurityAgency/ghidra) | Reverse engineering / decompilation platform |
| [Ps3GhidraScripts](https://github.com/clienthax/Ps3GhidraScripts) | A collection of scripts for parsing PS3 executables with Ghidra |
| [GhidraMCP](https://github.com/lauriewired/ghidramcp) | MCP Server for Ghidra |
| [rusty-psn](https://github.com/RainbowCookie32/rusty-psn) | A tool for downloading PS3 game updates |
| [RPCS3](https://rpcs3.net) | PlayStation 3 emulator and debugger |
| [Visual Studio Code](https://code.visualstudio.com) | Integrated development environment |
| [Skate3BlazeServer](https://github.com/skate6743/Skate3BlazeServer) | Custom Skate 3 Blaze Server |

### Ghidra Configuration

- Configure `Ghidra` to target the big-endian `PowerISA-Altivec-64-32addr` architecture.
- `Ps3GhidraScripts` was installed according to its `README.md` and `ppc_64_32.cspec` was updated in the process.
- `GhidraMCP` was configured to interface directly with the active `CodeBrowser` session for decompilation workflows.

### RPCS3 Configuration

- Enable debug mode.
- On the CPU tab, set `PPU Decoder` to `Interpreter (static)` to enable breakpoints.
- On the CPU tab, set `SPU XFloat Accuracy` to `Approximate XFloat`. Suggested by `Skate3BlazeServer`.
- On the Advanced tab, uncheck `Enable SPU Loop Detection`. Suggested by `Skate3BlazeServer`.

## Binary Preparation

Before being able to reverse engineer anything, the main executable must be extracted from the latest title update, which is version `1.05`.

1. The update package (`*.pkg`) was obtained via `rusty-psn` and unpacked to extract the `v1.05` executable (`EBOOT.BIN`).
2. This executable is slightly larger than the `v1.00` executable due to the addition of DLC.
3. The encrypted executable was decrypted into an ELF binary using RPCS3's built-in decryption utilities.
