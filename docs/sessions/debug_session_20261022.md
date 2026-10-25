# Debug Session 2026-10-22

I let Bob generate me the patch to inject the Gone Viral trophy hash.
Then in RPCS3 I disabled my other 2 debug pathes and only enabled the new patch.
Booted the game, set a breakpoint at `0x004bdf4c` to see the trophy id in r31.
Created a recording that shows after doing 3 manuals for mantastic voyage, the gone viral trophy unlocks.

<video width="100%" controls>
  <source src="../media/unlock_gone_viral_through_hash_trim.mp4" type="video/mp4">
  Your browser does not support the video tag.
</video>

## Summary and Next Steps
Patching in the gone viral hash and triggering it through the mantastic voyage requirements proves that the trophy hash determined for gone viral is correct and that `_opd_FUN_0046eb04` is a call site to unlock gone viral.
Maybe its the online call site, maybe not.
Next we need to understand what the xrefs of this function do.
We know gone viral is triggered through online play only, so figuring out where the netcode is in the eboot is a next step.
We can either trace up the call stack from this function, or find the entry point to the netcode and trace down to this function.
