# Debug Session 2026-10-18

I think this day on the weekend I learned about rpcs3 patches to change instruction streams in memory using `imported_patch.yml`.
I wrote two patches and added it to the repo here.
The goal was to improve the debug workflow. Until now I had to set a breakpoint, trigger a trophy (mantastic voyage) and then restart the game. A lot of overhead.
First patch was to not unlock the trophy and stub the `sceNpTrophyUnlockTrophy` function call. I proved with live debugger that the trophy no longer pops and doesn't show up in the rpcs3 profile anymore.
Problem was that triggering the same trophy for a second time, the trophy didn't try to unlock i.e. the trophy worker thread wasn't woken up.
I found some previous debug notes / AI inferred pseudo code here in the repo that mentioned that after `sceNpTrophyUnlockTrophy` is called, a local bitset vector gets updated with the trophy id. So it seemed like this bitset vector was used by the code that triggers a trophy to check if the trophy is already unlocked or not.
The second patch removes the bitset vector update and therefore trying again to earn the mantastic voyage trophy worked as expected and in the debugger I saw the trophy worker thread trigger the path.


# rpcs3 patch
unlike wiki rpcs3, this is the path i needed to install my custom patch in
```
ytbm@devbro:~/.var/app/net.rpcs3.RPCS3/config/rpcs3/patches$ ln -s /home/ytbm/Documents/ps3/skate-3-gone-viral/rpcs3/imported_patch.yml imported_patch.yml
```
