# Patch map

All 37 patches, in order. Apply them to Arch tag `v7.2.9-arch1`.

- **Default build** = patches `0001`–`0033`.
- **Candidate build** = patches `0001`–`0037`.

"Origin" tells you who wrote the code. Patches by other people are **unchanged** unless
the table says otherwise. Each patch has the exact source in its commit message.
Size is lines added / removed.

## A. Platform patches from t2linux

| # | What | Origin | Size |
|---|---|---|---|
| 0001 | Apple GMUX: allow switching to the integrated GPU at probe (amdgpu part only) | t2linux `2009` | +3/−1 |
| 0002 | Fix boot problems on the Radeon 5300 (power management, amdgpu part only) | t2linux `6001` | +16/−1 |
| 0003 | Reset the VI ASIC on MacBookPro15,1 (amdgpu part only). Harmless on Navi 10. t2linux has dropped it. **Does not apply on 7.2.10: skip it** | t2linux `7002` | +20/−1 |

## B. The GPU start-up problem

| # | What | Origin | Size |
|---|---|---|---|
| 0004 | If the first probe times out before a reset, defer the probe once. The retry works. **A fallback that never fires since `0032`.** Sent to amd-gfx on 2026-09-26 | Mine | +44/−1 |
| 0032 | Set up the display block before the power controller (SMU) on iMac20,x. This is the one `amdgpu_discovery.c` hunk of t2linux `7003`, with the missing `{ }` array terminator added | Atharva Tiwari (t2linux `7003`) | +28/−4 |
| 0033 | KFD: do not unlock on exit of a device that never initialised. A failed probe left a counter at −1 and caused a warning at the first resume | Mine | +8/−2 |

## C. The community stitch stack (unchanged)

Together these make one CRTC drive both tiles and show one 5120×2880 connector. They were
written for older iMacs (display engine DCE). On this machine (DCN) they need the
patches in group D.

| # | What | Origin | Size |
|---|---|---|---|
| 0005 | "Lean core": recognise the panel, wire the main and second link together, wake the second tile (`0x4F1`) | Pronkin `imac5k-lean-core-7.2.x` | +512/−6 |
| 0006 | "Stitch layer": one CRTC, two streams, a synthetic 5120×2880 mode, split planes, mirrored cursor. Adds `amdgpu.tiled_stitch` | Pronkin (erik2's stitch) | +1498/−11 |
| 0007 | Stop the resync worker at shutdown | Pronkin | +6/−2 |
| 0008 | Recover a lost tile link after a commit | Pronkin | +185/−8 |
| 0009 | Fix a `BUG_ON` in the "keep the live pair" shortcut | Pronkin | +12/−1 |
| 0010 | Resume: drop the cached second stream | Pronkin | +5/−1 |
| 0011 | Resume: arm the link-health check | Pronkin | +28/−1 |
| 0012 | Plain DP panel mode for the iMac Pro second tile, by panel ID | Pronkin | +38/−3 |
| 0013–0015 | Genlock fixes for older display engines. **Inactive** on this machine; kept so later patches apply | Pronkin | +74, +26, +16 |
| 0016 | A mode set postpones a queued post-commit pass | Pronkin | +13/−4 |
| 0017 | Genlock fixes for DCE 10/11. **Inactive** here; includes the measured re-align that `0025` turns on | br1uhnz | +177/−11 |
| 0018 | Never leave a stitched panel without modes | br1uhnz | +208/−36 |
| 0019 | Only split planes that the tiles can show | br1uhnz | +105/−13 |
| 0020 | Tile pairing, wake and recovery. **Includes re-arming `0x4F1` right after the main panel powers up** | br1uhnz | +222/−58 |
| 0021 | Bound the recovery escalation, fix an EDID swap race | br1uhnz | +7/−6 |

## D. Adaptation to the iMac20,x (DCN display engine)

| # | What | Origin | Size |
|---|---|---|---|
| 0022 | **Gate:** the tile code runs only on an iMac20,1 or iMac20,2 (DMI match). Other machines never enter it | Mine | +68/−2 |
| 0023 | Pair link 0 (eDP) with link 1 only, never a Thunderbolt port | Mine | +22/−1 |
| 0024 | Drive the second tile in plain DP mode for panel IDs `AE31`/`AE32` | Mine | +7/−1 |
| 0025 | Parameter `amdgpu.tiled_genlock`. `0` (default) aligns the tiles once and re-aligns if they drift. `1` resets per frame. **`1` is untested on hardware** | Mine | +26/−2 |
| 0026 | Per-frame tile reset on DCN (only used with `tiled_genlock=1`) | Mine | +29/−7 |
| 0027 | `APPLE5K:` log lines for bring-up | Mine | +75/−17 |

## E. Fixes that testing found

| # | What | Origin | Size |
|---|---|---|---|
| 0028 | Stage the second tile's cursor instead of programming it without the lock (found by code review before the first stitched boot) | Mine | +82/−31 |
| 0029 | Add the main stream before the second one on wake. Fixed a wake-up failure on every display wake | Mine | +18/−9 |
| 0030 | 20 GPU-scaled lower resolutions, with the panel timing unchanged | Mine | +187/−47 |
| 0031 | Fix a self-deadlock in the link-status escalation (found by a crash in run 057) | Mine | +1/−3 |

## F. Candidate patches (not in the default build)

| # | What | Origin | Size |
|---|---|---|---|
| 0034 | **L5:** if the second tile is lost, the kernel redoes the full mode set itself, with limits. Before, it asked the compositor, and Hyprland does not answer | Mine | +195/−27 |
| 0035 | **L6:** if that fails too, drive only the left tile and squeeze the desktop into it. The panel stretches it back. Sticky until reboot | Mine | +186/−12 |
| 0036 | `amdgpu.tiled_fault`: a test switch to **simulate** second-tile failures, because a real one cannot be provoked without opening the iMac | Mine | +43/−1 |
| 0037 | Do not retrain the second tile because of a short alignment blip after link training. The real failure is still caught by the link-health check | Mine | +18/−1 |

## Which patches could stand alone?

If you want to reuse something without the stitch layer, these are the small ones that do
not depend on it: `0032` (the same hunk as t2linux 7003), `0033` (KFD), and the idea in `0037`.
See [FOR-T2LINUX.md](FOR-T2LINUX.md).
