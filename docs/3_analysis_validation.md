# Analysis and Validation

Analysis and validation procedures confirming reverse engineering findings against live runtime execution.

## Trophy Details

PS3 games come with a `TROPHY.TRP` archive that describes every trophy the title can award.
Parsing this archive gives us the game-defined identifier for Gone Viral that we can then search for in the game executable.

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
| `hidden` | `yes` | Hidden in the XMB until earned |
| `ttype`  | `B`   | Bronze trophy |

Trophy ID 19 (`0x13`) is the unique identifier for Gone Viral within Skate 3.
This value is the handle the game engine uses when it calls the PS3 trophy service to award the trophy through a `sceNpTrophyUnlockTrophy` function call.
