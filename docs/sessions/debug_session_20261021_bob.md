# Debug Session 2026-10-21 — Hash→TrophyId Resolution & Gone Viral Hash Discovery

**Binary:** Skate 3 PS3 `v1.05` decrypted ELF  
**Tool:** RPCS3 debugger + Ghidra via GhidraMCP + RPCS3 memory viewer/searcher  
**Goal:** Understand how a 64-bit hash resolves to a numeric trophyId at runtime, then
use that knowledge to find the Gone Viral (`0x13`) trophy hash in memory.

---

## 1. Hash→TrophyId Resolution Confirmed (Mantastic Voyage)

Three breakpoints were set inside `skate3_themain_thread_trigger_trophy_hash`
(`0x004b11d4`) while triggering Mantastic Voyage (3+ manuals in a single line).

| BP | Address | Register | Value | Confirms |
|---|---|---|---|---|
| A | `0x004b12c4` | `r3` | `0x1bd48906e119a954` | Hash enters `FUN_004b10a0` intact |
| B | `0x004b12c8` | `r3` | `0xC` | `FUN_004b10a0` resolves hash → trophyId `0xC` |
| C | `0x004b12d0` | `r3`/`r4` | `0xC` / `0x0` | Resolved trophyId forwarded to `FUN_004a2e70` |

**Key finding:** The hash→trophyId mapping is **not a static table in the ELF**. It is a
runtime component/property registry. `FUN_004b10a0` uses:
- `FUN_0073f238(0x9cb9012080f727b9, hash)` — looks up the `TrophyDefinition` object by
  hash in the global component registry
- `FUN_0073a040(obj, 0xce07c47e53576879)` — reads the enabled flag property
- `FUN_0073a040(obj, 0x4042d2f96c4e4b8c)` — reads the trophyId integer property

The three 64-bit constants are property key hashes (component type, enabled flag, trophyId
field). The integer `0xC` lives in a runtime data object populated from game data at boot.

### Where `0x4042d2f96c4e4b8c` comes from

This constant was not discovered at runtime — it was read directly from the **Ghidra
decompilation of `FUN_004b10a0`**. Decompiling that function produced:

```c
undefined4 FUN_004b10a0(undefined8 param_1)
{
  ...
  uVar1 = FUN_0073f238(0x9cb9012080f727b9, param_1);  // find TrophyDefinition by hash
  FUN_0073a568(local_30, uVar1, 0);
  if (local_30[0] != 0) {
    pcVar2 = (char *)FUN_0073a040(local_30, 0xce07c47e53576879, 0); // read "enabled" flag
    ...
    if (*pcVar2 != '\0') {
      puVar3 = (undefined4 *)FUN_0073a040(local_30, 0x4042d2f96c4e4b8c, 0); // read "trophyId"
      ...
      uVar4 = *puVar3;   // dereference → the integer trophyId
    }
  }
  return uVar4;
}
```

Ghidra embedded the constant `0x4042d2f96c4e4b8c` as a literal in the decompiled output
because the PPC disassembly builds it from immediate instructions (`lis`, `ori`,
`rldicr`, `oris`, `ori`) — the same pattern used to build the trophy hash constants in
`FUN_00500734`. Ghidra reassembles these into a single 64-bit immediate in the
decompiler output.

This constant is the **property key hash for the "trophyId" field** inside every
`TrophyDefinition` object in the component registry. It is the hashed form of whatever
string the game engine uses internally to name that field (e.g. `"trophyId"` or
`"award_id"` — the original string is not present in the binary). Every property in the
system is identified by its hash, not its name.

Because the property key is the same for every trophy's definition object, we could use
it as a search anchor: any memory location containing `40 42 d2 f9 6c 4e 4b 8c` followed
by a non-zero 4-byte integer is a trophyId property entry. Searching for
`40 42 d2 f9 6c 4e 4b 8c 00 00 00 13` directly located the Gone Viral entry.

---

## 2. All 21 Callers of `FUN_004b11d4` Enumerated

All 21 call sites were decompiled. None carry a hardcoded constant that is obviously
`0x13`. However, this does **not** mean Gone Viral has no hash-based path — any of the
19 distinct 64-bit hash constants could resolve to `0x13` via the runtime registry.

**Correction to prior session:** the earlier conclusion that Gone Viral only uses the
`0x7b` network path was premature. The registry is runtime data; static analysis alone
cannot determine which hash maps to which trophyId.

---

## 3. TrophyDefinition Object Layout Discovered in Memory

Using BP `0x004b1190` (inside `FUN_004b10a0`, at the trophyId property read call) during
Mantastic Voyage trigger:

- **`r3` = `0xd0068720`** → `*(r3)` = **`0x30978ec0`** = `OBJ_A` (Mantastic Voyage node)
- After `FUN_0073a040` returns: **`r3` = `0x30903008`** → `*(r3)` = `0x0000000C` (trophyId)

Searching memory for the Mantastic Voyage hash `1b d4 89 06 e1 19 a9 54` in `0x309xxxxx`:
- `0x309282e0` — bucket array slot
- `0x30978ed8` — inside node at `OBJ_A + 0x18`

Node layout confirmed by comparing a second trophy (`0xb598d5bd5d16ec4f`, score > 9,999):

```
node + 0x00  : uint32  pointer to property data block
node + 0x04  : uint32  0x00000001  (ref count)
node + 0x08  : uint32  0x00020002  (schema descriptor)
node + 0x0c  : uint32  0x00000000  (padding)
node + 0x10  : uint64  0x0000000000000000
node + 0x18  : uint64  trophy hash  ← 64-bit hash key
```

Property data block layout (confirmed across all five trophies):
```
block + 0x00  : uint64  0x4042d2f96c4e4b8c  (property key hash for "trophyId")
block + 0x08  : uint32  trophyId integer
block + 0x0c  : ...
```

---

## 4. All Known Trophy Hashes & IDs

### Hashes confirmed via memory search (node + property block located at runtime)

These five were found by searching for their 8-byte hash in the `0x309xxxxx` heap region,
then reading `node+0x00` (property block pointer) and `property_block+0x08` (trophyId).

| Hash | Node base | Property block | trophyId (hex) | trophyId (dec) | Trophy name |
|---|---|---|---|---|---|
| `0xe9c379e93fa2693f` | `0x30978e60` | `0x30902fc0` | `0x0A` | 10 | unknown |
| `0x1bd48906e119a954` | `0x30978ec0` | `0x30903000` | `0x0C` | 12 | Mantastic Voyage |
| `0xd127d95b8d349b72` | `0x30978f50` | `0x30903060` | `0x0B` | 11 | unknown |
| `0x2a9d5f12bc41ee19` | `0x30979100` | `0x30903180` | `0x07` | 7 | Trick Spammer (30,000 pt line) |
| `0xb598d5bd5d16ec4f` | `0x30979490` | `0x309033c0` | `0x08` | 8 | unknown |
| `0x17e8ac1014ab97a5` | `0x309791f0` | `0x30903220` | `0x13` | 19 | **Gone Viral** |

Property blocks are **not** in sequential trophyId order and have variable-size gaps —
arithmetic extrapolation to find a given trophyId is not reliable.

### Additional hashes found via static analysis (caller decompilation only, trophyId unconfirmed at runtime)

These constants appear as hardcoded arguments to `skate3_themain_thread_trigger_trophy_hash`
across the 21 callers. Their trophyIds were **not** confirmed in memory — they may or may
not map to the values listed.

| Hash | Caller | Trigger condition | trophyId (runtime unconfirmed) |
|---|---|---|---|
| `0x1bd48906e119a954` | `skate3_themain_thread_evaluate_trick_line` | manual count `struct+0x167e > 2` | `0x0C` — **confirmed** |
| `0xb598d5bd5d16ec4f` | `skate3_themain_thread_evaluate_trick_line` | score > 9,999 | `0x08` — **confirmed** |
| `0x2a9d5f12bc41ee19` | `skate3_themain_thread_evaluate_trick_line` | score > 29,999 | `0x07` — **confirmed** |
| `0xe9c379e93fa2693f` | `skate3_themain_thread_evaluate_trick_line` | trick count > 2 | `0x0A` — **confirmed** |
| `0xd127d95b8d349b72` | `skate3_themain_thread_evaluate_trick_line` | float threshold (grind/speed) | `0x0B` — **confirmed** |
| `0x17e8ac1014ab97a5` | `_opd_FUN_0046eb04` | event type `0x2b`, `piVar1+0x4c != 0` | `0x13` — **confirmed** |
| `0x707b5aac11926cc8` | `_opd_FUN_0045e358` | `param_1+0x68` and `+0x69` flags | unconfirmed |
| `0x1e9eac0fdd0c4fe4` | `_opd_FUN_004b74d0` | `param_1+0x1c8 != 0`, list count > 1 | unconfirmed |
| `0x111ab31ff5c190ea` | `FUN_004fffb4` | `param_2 == 0xfc`, positive count at `param_4+0x3c+0x2674` | unconfirmed |
| `0xbaf06f08fcc02187` | `FUN_00502588` | time threshold | unconfirmed |
| `0xdcc17bf0811e9496` | `FUN_00502588` | score > 74,999 | unconfirmed |
| `0x301ee029d774cd29` | `_opd_FUN_00586b34` | `FUN_0057de20` return > `0x3a97` | unconfirmed |
| `0x82ffb228ffb0179b` | `_opd_FUN_00586b34` | `FUN_0057de20` return > `0xc34f` | unconfirmed |
| `0xb10e354d25008f04` | `_opd_FUN_00586b34` | `FUN_0057de20` return > `0x1869f` | unconfirmed |
| `0x7ac6c1724422f4dc` | `_opd_FUN_00586b34` | `FUN_0057de20` return > `0x493df` | unconfirmed |
| `0x325fc011e85b6ff9` | `_opd_FUN_00586b34` | `FUN_0057de20` return > `0x927bf` | unconfirmed |
| `0xc4ab2f5465ed925d` | `_opd_FUN_00586b34` | `FUN_0057de20` return > `0xf423f` | unconfirmed |
| `0x6dd8d776a59cba15` | `FUN_00467a38` | `param_1+400 == 0x16` and `local_c0 == 6` | unconfirmed |
| `0xe760bd40bbcd8fc1` | `FUN_0057e004` | `FUN_00f308c0(...,1) > 0x4f` | unconfirmed |
| dynamic | `_opd_FUN_0062862c` | hash resolved at runtime from registry | unconfirmed |
| unknown | `FUN_00f258e0` | OPD thunk — hash set by caller | unconfirmed |

---

## 5. Gone Viral TrophyDefinition Found in Memory

Instead of walking property blocks by stride, searched directly for the 12-byte sequence:

```
40 42 d2 f9  6c 4e 4b 8c  00 00 00 13
```
(property key hash `0x4042d2f96c4e4b8c` followed immediately by trophyId `0x13`)

**Result: found at `0x30903220`** — one hit, the Gone Viral property block.

Searched for the 4-byte pointer `30 90 32 20` to find the owning node:  
**Found at `0x309791f0`** — the Gone Viral `TrophyDefinition` node.

Node at `0x309791f0`:
```
+0x00: 30903220  → property block (trophyId = 0x13) ✓
+0x04: 00000001
+0x08: 00020002
+0x0c: 00000000
+0x10: 00000000
+0x14: 00000000
+0x18: 17e8ac10 14ab97a5  ← Gone Viral hash
```

---

## 6. Gone Viral Hash: `0x17e8ac1014ab97a5`

This hash was already present in the 21-caller enumeration — it is the constant passed
to `skate3_themain_thread_trigger_trophy_hash` in **`_opd_FUN_0046eb04`** (`0x0046eb04`):

```c
if (*(char *)(piVar1 + 0x4c) != '\0') {
    skate3_themain_thread_trigger_trophy_hash(0x17e8ac1014ab97a5, 0);
    ...
}
```

**Gone Viral has a hash-based trigger path through `FUN_004b11d4`**, separate from and
in addition to the `0x7b` network path through `FUN_00f2deb0`. The earlier conclusion
that the network path was the only route was wrong.

---

## 7. Summary of Known Gone Viral Trigger Paths

| Path | Entry point | How it reaches the worker |
|---|---|---|
| **Hash path** (newly confirmed) | `_opd_FUN_0046eb04` (`0x0046eb04`) with hash `0x17e8ac1014ab97a5` | → `FUN_004b11d4` → `FUN_004b10a0` resolves to `0x13` → `FUN_004a2e70` → `_opd_FUN_004acbc8` → worker |
| **Network path** (from static analysis, unconfirmed at runtime) | `NetEvent_HandlePeerHandshake_GoneViralCheck` (`0x0045dc58`) on event `0x7b` | → `FUN_00f2deb0(ctx, 0x13)` → unknown queue path |

---

## 8. Next Steps

- Decompile `_opd_FUN_0046eb04` in detail — understand what game event triggers it and
  what `*(char *)(piVar1 + 0x4c) != '\0'` means (the condition that fires Gone Viral)
- Set BP at `0x004b12c8` and trigger the `_opd_FUN_0046eb04` path to confirm
  `r3 = 0x13` at runtime, proving the hash-based path works end-to-end
- Determine the relationship between the hash path and the network path — are they
  redundant, or does one set a flag that the other checks?
