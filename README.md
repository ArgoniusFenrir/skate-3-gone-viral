# Skate 3 Gone Viral

The Gone Viral trophy in Skate 3 (PS3) has been broken since May 2025.
The trophy no longer unlocks and the [EA forum thread](https://forums.ea.com/discussions/skate-franchise-discussion-en/gone-viral-trophy-on-skate-3-ps3/12244224) on the issue has seen little to no progress from EA's side.

This repo documents reverse engineering efforts on how to trigger the trophy unlock, with the goal of either producing a community-driven open source fix or providing EA with enough detail to ship an official fix.

## Documentation

Documentation of the investigation, reverse engineering, and analysis of the Gone Viral trophy can be found [here](docs/README.md).

## Tools

| Tool | Purpose |
|------|---------|
| [Ghidra](https://github.com/NationalSecurityAgency/ghidra) | Reverse engineering / decompilation platform |
| [Ps3GhidraScripts](https://github.com/clienthax/Ps3GhidraScripts) | A collection of scripts for parsing PS3 executables with Ghidra |
| [GhidraMCP](https://github.com/lauriewired/ghidramcp) | MCP Server for Ghidra |
| [rusty-psn](https://github.com/RainbowCookie32/rusty-psn) | A tool for downloading PS3 game updates |
| [RPCS3](https://rpcs3.net) | PlayStation 3 emulator and debugger |

## Community & Discussion

| Place | Link |
|-------|------|
| EA Forums thread | [Gone Viral Trophy on Skate 3 PS3](https://forums.ea.com/discussions/skate-franchise-discussion-en/gone-viral-trophy-on-skate-3-ps3/12244224) |
| PSN Profiles thread | [Gone Viral](https://forum.psnprofiles.com/topic/178600-gone-viral) |

## Contributing

If you have reverse engineering findings of your own, captured network traffic, etc., please open a PR or an issue.
Contributions and collaboration are much appreciated.
