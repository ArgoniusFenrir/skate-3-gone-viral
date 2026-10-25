# Debug Session 2026-10-21

Updated runtime confirmation — exact addresses
BP A — 0x004b12c4 (call to hash→ID resolver)
004b12c4: bl 0x004b10a0

Check: r3 = 0x1bd48906e119a954 — the hash is about to be looked up.

BP B — 0x004b12c8 (first instruction after FUN_004b10a0 returns) ← key
004b12c8: rldicl r4,r27,0x0,0x20   ← first instruction after bl returns
004b12cc: extsw  r3,r3              ← sign-extends the return value (trophyId)

Check: r3 = 0xC — this is the resolved trophyId returned by FUN_004b10a0. This single breakpoint directly confirms the hash → 0xC mapping.

BP C — 0x004b12d0 (call to FUN_004a2e70 with resolved ID)
004b12d0: bl 0x004a2e70

Check: r3 = 0xC, r4 = 0x0 — confirms 0xC is passed forward as the numeric trophyId into the enqueue chain.

So the one breakpoint you need is 0x004b12c8: trigger Mantastic Voyage (3 manuals), and when it fires, r3 = 0xC confirms the entire hash→ID resolution chain.

=======

debugger results:
- bp A ; r3 : 0x1bd48906e119a954
- bp B ; r3 : 0xc
- bp C ; r3 : 0xc and r4 : 0x0

=====

Plan: Find all TrophyDefinition objects in memory and locate the one with trophyId 0x13

Step 1 — Catch a live TrophyDefinition object pointer

Trigger Mantastic Voyage (3 manuals) with a breakpoint at 0x004b1190 — the bl 0x0073a040 call that reads the trophyId property.

When the BP fires:

r3 = pointer to local_30 (the iterator struct on stack)
local_30[0] = *(r3 + 0) — this is the pointer to the live TrophyDefinition object for Mantastic Voyage
Read *(r3) from the memory viewer. Call this address OBJ_A.

```
r0 : 0x1
r1 : 0xd00686b0 -> 00 00 00 00 d0 06 87 50
r2 : 0x1842b50  -> 01 4e 11 40 01 4e 11 60
r3 : 0xd0068720 -> 30 97 8e c0 00 00 00 00
r4 : 0x4042d2f96c4e4b8c
r5 : 0x0
r6 : 0x30903000 -> 40 42 d2 f9 6c 4e 4b 8c
r7 : 0x2
r8 : 0x30903010 -> ce 07 c4 7e 53 57 68 79
r9 : 0x30903018 -> 01 00 00 00 00 0a 00 c0
r10: 0x30978ec4 -> 00 00 00 03 00 02 00 02
r11: 0x309000a0 -> 00 00 00 00 00 00 00 00
r12: 0x24004042
r13: 0x10007be0 -> 00 00 00 00 00 00 00 00
r14: 0x0
r15: 0x0
r16: 0x0
r17: 0x0
r18: 0x0
r19: 0x0
r20: 0x0
r21: 0x0
r22: 0xffffffffffffffff
r23: 0x20000000 -> 00 00 00 00 00 00 00 00
r24: 0x8c
r25: 0x5
r26: 0x8c
r27: 0x30051d20 -> 01 7b 06 b0 00 00 00 00
r28: 0x0
r29: 0xd0068720 -> 30 97 8e c0 00 00 00 00
r30: 0x0
r31: 0xd0068720 -> 30 97 8e c0 00 00 00 00
CR: 0x24004044, CR7: [LT=0 GT=1 EQ=0 SO=0]
f0 : 100          [0x4059000000000000] (f32=0x42c80000)
f1 : 3.91667      [0x400f555580000000] (f32=0x407aaaac)
f2 : 1            [0x3ff0000000000000] (f32=0x3f800000)
f3 : 0.351831     [0x3fd6846560000000] (f32=0x3eb4232b)
f4 : 0.224529     [0x3fccbd5d40000000] (f32=0x3e65eaea)
f5 : -1           [0xbff0000000000000] (f32=0xbf800000)
f6 : 1            [0x3ff0000000000000] (f32=0x3f800000)
f7 : 0.141457     [0x3fc21b4600000000] (f32=0x3e10da30)
f8 : 0.999863     [0x3feffee0e0000000] (f32=0x3f7ff707)
f9 : r0 : 0x1
r1 : 0xd00686b0 -> 00 00 00 00 d0 06 87 50
r2 : 0x1842b50  -> 01 4e 11 40 01 4e 11 60
r3 : 0xd0068720 -> 30 97 8e c0 00 00 00 00
r4 : 0x4042d2f96c4e4b8c
r5 : 0x0
r6 : 0x30903000 -> 40 42 d2 f9 6c 4e 4b 8c
r7 : 0x2
r8 : 0x30903010 -> ce 07 c4 7e 53 57 68 79
r9 : 0x30903018 -> 01 00 00 00 00 0a 00 c0
r10: 0x30978ec4 -> 00 00 00 03 00 02 00 02
r11: 0x309000a0 -> 00 00 00 00 00 00 00 00
r12: 0x24004042
r13: 0x10007be0 -> 00 00 00 00 00 00 00 00
r14: 0x0
r15: 0x0
r16: 0x0
r17: 0x0
r18: 0x0
r19: 0x0
r20: 0x0
r21: 0x0
r22: 0xffffffffffffffff
r23: 0x20000000 -> 00 00 00 00 00 00 00 00
r24: 0x8c
r25: 0x5
r26: 0x8c
r27: 0x30051d20 -> 01 7b 06 b0 00 00 00 00
r28: 0x0
r29: 0xd0068720 -> 30 97 8e c0 00 00 00 00
r30: 0x0
r31: 0xd0068720 -> 30 97 8e c0 00 00 00 00
CR: 0x24004044, CR7: [LT=0 GT=1 EQ=0 SO=0]
f0 : 100          [0x4059000000000000] (f32=0x42c80000)
f1 : 3.91667      [0x400f555580000000] (f32=0x407aaaac)
f2 : 1            [0x3ff0000000000000] (f32=0x3f800000)
f3 : 0.351831     [0x3fd6846560000000] (f32=0x3eb4232b)
f4 : 0.224529     [0x3fccbd5d40000000] (f32=0x3e65eaea)
f5 : -1           [0xbff0000000000000] (f32=0xbf800000)
f6 : 1            [0x3ff0000000000000] (f32=0x3f800000)
f7 : 0.141457     [0x3fc21b4600000000] (f32=0x3e10da30)
f8 : 0.999863     [0x3feffee0e0000000] (f32=0x3f7ff707)
f9 : 0.324901     [0x3fd4cb2c40000000] (f32=0x3ea65962)
f10: 0.224529     [0x3fccbd5d40000000] (f32=0x3e65eaea)
f11: 0            [                  ] (f32=0x0)
f12: 0.0443022    [0x3fa6aec8a0000000] (f32=0x3d357645)
f13: 0            [                  ] (f32=0x0)
f14: 0            [                  ] (f32=0x0)
f15: 0            [                  ] (f32=0x0)
f16: 0            [                  ] (f32=0x0)
f17: 0            [                  ] (f32=0x0)
f18: 0            [                  ] (f32=0x0)
f19: 0            [                  ] (f32=0x0)
f20: 0            [                  ] (f32=0x0)
f21: 0            [                  ] (f32=0x0)
f22: 0            [                  ] (f32=0x0)
f23: 0            [                  ] (f32=0x0)
f24: 0            [                  ] (f32=0x0)
f25: 0            [                  ] (f32=0x0)
f26: 0            [                  ] (f32=0x0)
f27: 0            [                  ] (f32=0x0)
f28: 0            [                  ] (f32=0x0)
f29: 0            [                  ] (f32=0x0)
f30: 0            [                  ] (f32=0x0)
f31: 0.0166667    [0x3f91111120000000] (f32=0x3c888889)
v0 : 439c6064 4293e669 c3cfb104 3e02aa54 [x: 312.753 y: 73.95 z: -415.383 w: 0.127603]
v1 : 32656ae0 [x: 1.33539e-08]
v2 : 3ae869d7 [x: 0.00177317]
v3 : 3f800000 [x: 1]
v4 : 00000000 [x: 0]
v5 : 00000000 [x: 0]
v6 : 00000000 [x: 0]
v7 : 00000000 [x: 0]
v8 : 3f800000 [x: 1]
v9 : 41b6438a [x: 22.783]
v10: 00000000 [x: 0]
v11: 41b6438a [x: 22.783]
v12: 41b6438a [x: 22.783]
v13: 32656ae0 [x: 1.33539e-08]
v14: 3db63a9e [x: 0.088979]
v15: be5bbfca [x: -0.214599]
v16: 3f800000 [x: 1]
v17: 40490fdb [x: 3.14159]
v18: 0000001f [x: 4.34403e-44]
v19: 04050607 08090a0b 00010203 0c0d0e0f [x: 1.56368e-36 y: 4.12387e-34 z: 9.25572e-41 w: 1.08665e-31]
v20: 00000000 [x: 0]
v21: 00000000 [x: 0]
v22: 00000000 [x: 0]
v23: 00000000 [x: 0]
v24: 00000000 [x: 0]
v25: 00000000 [x: 0]
v26: 00000000 [x: 0]
v27: 00000000 [x: 0]
v28: 00000000 [x: 0]
v29: 00000000 [x: 0]
v30: 00000000 [x: 0]
v31: 00000000 [x: 0]
CIA: 0x4b1190
LR: 0x4b1150
CTR: 0x0
VRSAVE: 0xffffffff
XER: [CA=0 | OV=0 | SO=0 | CNT=0]
VSCR: [SAT=140230258268480 | NJ=1]
FPSCR: [FL=1 | FG=0 | FE=0 | FU=0]
Reservation Addr: none
Reservation Data (entire cache line):
[0x00] 00000000 00000001 00000000 00000001
[0x10] 00000000 00000001 ffffffff ffffffff
[0x20] 00000000 00000001 ffffffff 00000004
[0x30] 000196a5 00000000 428e2ff0 428e3650
[0x40] 30903000 00000002 ffffffff ffffffff
[0x50] ffffffff ffffffff 00000000 00000000
[0x60] 309da020 00000002 428e3320 428e2ee0
[0x70] 00000550 0001eae0 ffffffff ffffffff0.324901     [0x3fd4cb2c40000000] (f32=0x3ea65962)
f10: 0.224529     [0x3fccbd5d40000000] (f32=0x3e65eaea)
f11: 0            [                  ] (f32=0x0)
f12: 0.0443022    [0x3fa6aec8a0000000] (f32=0x3d357645)
f13: 0            [                  ] (f32=0x0)
f14: 0            [                  ] (f32=0x0)
f15: 0            [                  ] (f32=0x0)
f16: 0            [                  ] (f32=0x0)
f17: 0            [                  ] (f32=0x0)
f18: 0            [                  ] (f32=0x0)
f19: 0            [                  ] (f32=0x0)
f20: 0            [                  ] (f32=0x0)
f21: 0            [                  ] (f32=0x0)
f22: 0            [                  ] (f32=0x0)
f23: 0            [                  ] (f32=0x0)
f24: 0            [                  ] (f32=0x0)
f25: 0            [                  ] (f32=0x0)
f26: 0            [                  ] (f32=0x0)
f27: 0            [                  ] (f32=0x0)
f28: 0            [                  ] (f32=0x0)
f29: 0            [                  ] (f32=0x0)
f30: 0            [                  ] (f32=0x0)
f31: 0.0166667    [0x3f91111120000000] (f32=0x3c888889)
v0 : 439c6064 4293e669 c3cfb104 3e02aa54 [x: 312.753 y: 73.95 z: -415.383 w: 0.127603]
v1 : 32656ae0 [x: 1.33539e-08]
v2 : 3ae869d7 [x: 0.00177317]
v3 : 3f800000 [x: 1]
v4 : 00000000 [x: 0]
v5 : 00000000 [x: 0]
v6 : 00000000 [x: 0]
v7 : 00000000 [x: 0]
v8 : 3f800000 [x: 1]
v9 : 41b6438a [x: 22.783]
v10: 00000000 [x: 0]
v11: 41b6438a [x: 22.783]
v12: 41b6438a [x: 22.783]
v13: 32656ae0 [x: 1.33539e-08]
v14: 3db63a9e [x: 0.088979]
v15: be5bbfca [x: -0.214599]
v16: 3f800000 [x: 1]
v17: 40490fdb [x: 3.14159]
v18: 0000001f [x: 4.34403e-44]
v19: 04050607 08090a0b 00010203 0c0d0e0f [x: 1.56368e-36 y: 4.12387e-34 z: 9.25572e-41 w: 1.08665e-31]
v20: 00000000 [x: 0]
v21: 00000000 [x: 0]
v22: 00000000 [x: 0]
v23: 00000000 [x: 0]
v24: 00000000 [x: 0]
v25: 00000000 [x: 0]
v26: 00000000 [x: 0]
v27: 00000000 [x: 0]
v28: 00000000 [x: 0]
v29: 00000000 [x: 0]
v30: 00000000 [x: 0]
v31: 00000000 [x: 0]
CIA: 0x4b1190
LR: 0x4b1150
CTR: 0x0
VRSAVE: 0xffffffff
XER: [CA=0 | OV=0 | SO=0 | CNT=0]
VSCR: [SAT=140230258268480 | NJ=1]
FPSCR: [FL=1 | FG=0 | FE=0 | FU=0]
Reservation Addr: none
Reservation Data (entire cache line):
[0x00] 00000000 00000001 00000000 00000001
[0x10] 00000000 00000001 ffffffff ffffffff
[0x20] 00000000 00000001 ffffffff 00000004
[0x30] 000196a5 00000000 428e2ff0 428e3650
[0x40] 30903000 00000002 ffffffff ffffffff
[0x50] ffffffff ffffffff 00000000 00000000
[0x60] 309da020 00000002 428e3320 428e2ee0
[0x70] 00000550 0001eae0 ffffffff ffffffff
```

Step 2 — Find the trophyId field offset within the object
At BP 0x004b1190, FUN_0073a040 is about to read property 0x4042d2f96c4e4b8c (the trophyId) from the object and return a pointer to it. After it returns (next BP at 0x004b1194):

r3 = pointer to the trophyId integer field inside OBJ_A
So: trophyId_field_offset = r3 - OBJ_A

Now you know: for any TrophyDefinition object, the trophyId integer is at object_base + trophyId_field_offset.

Confirm: *(r3) = 0xC — that's the Mantastic Voyage trophyId.

bp 0x004b1194
```
r0 : 0x4b1194   -> nop
r1 : 0xd00686b0 -> 00 00 00 00 d0 06 87 50
r2 : 0x1842b50  -> 01 4e 11 40 01 4e 11 60
r3 : 0x30903008 -> 00 00 00 0c 00 06 00 c0
r4 : 0x30978ec0 -> 30 90 30 00 00 00 00 03
r5 : 0x4042d2f96c4e4b8c
r6 : 0x30903000 -> 40 42 d2 f9 6c 4e 4b 8c
r7 : 0x2
r8 : 0x30903000 -> 40 42 d2 f9 6c 4e 4b 8c
r9 : 0x1
r10: 0x30978ec4 -> 00 00 00 03 00 02 00 02
r11: 0x309000a0 -> 00 00 00 00 00 00 00 00
r12: 0x24004042
r13: 0x10007be0 -> 00 00 00 00 00 00 00 00
r14: 0x0
r15: 0x0
r16: 0x0
r17: 0x0
r18: 0x0
r19: 0x0
r20: 0x0
r21: 0x0
r22: 0xffffffffffffffff
r23: 0x20000000 -> 00 00 00 00 00 00 00 00
r24: 0x8c
r25: 0x5
r26: 0x8c
r27: 0x30051d20 -> 01 7b 06 b0 00 00 00 00
r28: 0x0
r29: 0xd0068720 -> 30 97 8e c0 00 00 00 00
r30: 0x0
r31: 0xd0068720 -> 30 97 8e c0 00 00 00 00
CR: 0x24004042, CR7: [LT=0 GT=0 EQ=1 SO=0]
f0 : 100          [0x4059000000000000] (f32=0x42c80000)
f1 : 3.91667      [0x400f555580000000] (f32=0x407aaaac)
f2 : 1            [0x3ff0000000000000] (f32=0x3f800000)
f3 : 0.351831     [0x3fd6846560000000] (f32=0x3eb4232b)
f4 : 0.224529     [0x3fccbd5d40000000] (f32=0x3e65eaea)
f5 : -1           [0xbff0000000000000] (f32=0xbf800000)
f6 : 1            [0x3ff0000000000000] (f32=0x3f800000)
f7 : 0.141457     [0x3fc21b4600000000] (f32=0x3e10da30)
f8 : 0.999863     [0x3feffee0e0000000] (f32=0x3f7ff707)
f9 : 0.324901     [0x3fd4cb2c40000000] (f32=0x3ea65962)
f10: 0.224529     [0x3fccbd5d40000000] (f32=0x3e65eaea)
f11: 0            [                  ] (f32=0x0)
f12: 0.0443022    [0x3fa6aec8a0000000] (f32=0x3d357645)
f13: 0            [                  ] (f32=0x0)
f14: 0            [                  ] (f32=0x0)
f15: 0            [                  ] (f32=0x0)
f16: 0            [                  ] (f32=0x0)
f17: 0            [                  ] (f32=0x0)
f18: 0            [                  ] (f32=0x0)01 4b 00 78  01 4b 00 90  01 4b 00 a8  01 4b 00 c0
f19: 0            [                  ] (f32=0x0)
f20: 0            [                  ] (f32=0x0)
f21: 0            [                  ] (f32=0x0)
f22: 0            [                  ] (f32=0x0)
f23: 0            [                  ] (f32=0x0)
f24: 0            [                  ] (f32=0x0)
f25: 0            [                  ] (f32=0x0)
f26: 0            [                  ] (f32=0x0)
f27: 0            [                  ] (f32=0x0)
f28: 0            [                  ] (f32=0x0)
f29: 0            [                  ] (f32=0x0)
f30: 0            [                  ] (f32=0x0)
f31: 0.0166667    [0x3f91111120000000] (f32=0x3c888889)
v0 : 439c6064 4293e669 c3cfb104 3e02aa54 [x: 312.753 y: 73.95 z: -415.383 w: 0.127603]
v1 : 32656ae0 [x: 1.33539e-08]
v2 : 3ae869d7 [x: 0.00177317]
v3 : 3f800000 [x: 1]
v4 : 00000000 [x: 0]
v5 : 00000000 [x: 0]
v6 : 00000000 [x: 0]
v7 : 00000000 [x: 0]
v8 : 3f800000 [x: 1]
v9 : 41b6438a [x: 22.783]
v10: 00000000 [x: 0]
v11: 41b6438a [x: 22.783]
v12: 41b6438a [x: 22.783]
v13: 32656ae0 [x: 1.33539e-08]
v14: 3db63a9e [x: 0.088979]
v15: be5bbfca [x: -0.214599]
v16: 3f800000 [x: 1]
v17: 40490fdb [x: 3.14159]
v18: 0000001f [x: 4.34403e-44]
v19: 04050607 08090a0b 00010203 0c0d0e0f [x: 1.56368e-36 y: 4.12387e-34 z: 9.25572e-41 w: 1.08665e-31]
v20: 00000000 [x: 0]
v21: 00000000 [x: 0]
v22: 00000000 [x: 0]
v23: 00000000 [x: 0]
v24: 00000000 [x: 0]
v25: 00000000 [x: 0]
v26: 00000000 [x: 0]
v27: 00000000 [x: 0]
v28: 00000000 [x: 0]
v29: 00000000 [x: 0]
v30: 00000000 [x: 0]
v31: 00000000 [x: 0]
CIA: 0x4b1194
LR: 0x4b1194
CTR: 0x0
VRSAVE: 0xffffffff
XER: [CA=0 | OV=0 | SO=0 | CNT=0]
VSCR: [SAT=140230258268480 | NJ=1]
FPSCR: [FL=1 | FG=0 | FE=0 | FU=0]
Reservation Addr: none
Reservation Data (entire cache line):
[0x00] 00000000 00000001 00000000 00000001
[0x10] 00000000 00000001 ffffffff ffffffff
[0x20] 00000000 00000001 ffffffff 00000004
[0x30] 000196a5 00000000 428e2ff0 428e3650
[0x40] 30903000 00000002 ffffffff ffffffff
[0x50] ffffffff ffffffff 00000000 00000000
[0x60] 309da020 00000002 428e3320 428e2ee0
[0x70] 00000550 0001eae0 ffffffff ffffffff
```

at bp 0x004b1194 and memory address 0x01834ae0 i see in memory viewer:
01 4b 00 78  01 4b 00 90  01 4b 00 a8  01 4b 00 c0


-----

memory viewer ; search hash `1bd48906e119a954` ; log shows:
```
S GUI: Found at 0x309282e0
S GUI: Found at 0x30978ed8
S GUI: Search completed (found 2 matches)
```

memory searcher `0xb598d5bd5d16ec4f`:
```
S GUI: Found at 0x30928290
S GUI: Found at 0x309794a8
S GUI: Search completed (found 2 matches)
```

======

many memory viewer and searcher queries together with Bob ...

=====

Bob:

The Gone Viral trophy hash is 0x17e8ac1014ab97a5.

Reading the full node layout:

0x309791f0  +0x00: 30903220  → property block (trophyId = 0x13) ✓
            +0x04: 00000001  → ref count
            +0x08: 00020002  → schema descriptor
            +0x0c: 00000000  → padding
            +0x10: 00000000  → (continuation)
            +0x14: 00000000  → (continuation)
            +0x18: 17e8ac1014ab97a5  ← Gone Viral hash

And we already saw this hash — 0x17e8ac1014ab97a5 is the exact constant passed to skate3_themain_thread_trigger_trophy_hash in _opd_FUN_0046eb04:

if (*(char *)(piVar1 + 0x4c) != '\0') {
    skate3_themain_thread_trigger_trophy_hash(0x17e8ac1014ab97a5, 0);

Gone Viral does have a hash-based trigger path through FUN_004b11d4, and it's _opd_FUN_0046eb04. The 0x7b network path through FUN_00f2deb0 is a second, separate path.

The next step is to understand what _opd_FUN_0046eb04 actually is — what game event triggers it, and what *(char *)(piVar1 + 0x4c) != '\0' means. That's the condition that fires Gone Viral through the engine's normal trophy system. Want to dig into that function now?
