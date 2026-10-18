# Run journal

Every test boot I ran on the iMac20,2 for this work, including the failures. Newest at the
bottom. Read it top to bottom to see how the work developed.

How to read it:

- **Run numbers** come from my private test rig (not published). A run is one boot of a
  test module. The raw logs stay on my machine. This table is the summary I wrote at the
  time.
- **"Me: A"** is my own visual grade of what I saw on the panel. A means clean.
- **Build names and patch numbers.** Early rows say "PoC" plus a patch number. They are
  older builds with fewer patches. Rows from 2026-10-11 use the **old numbering**: the
  patch that is `0027` today was `0024` then, because the series gained three patches in
  front of it on 2026-10-12. Rows from 2026-10-12 use today's numbers. Rows that say
  `bcea67d6…` are the **candidate** build (see the README).
- **`stitch-full`** is my automated check: boot checks, then three display off/on cycles
  and a 2560×1440 round trip, all judged from the kernel log.
- **`imachy`** is the name of my iMac.

| Run | Date | What | Result |
|---|---|---|---|
| 001 | 2026-10-11 | Dry run: arm + disarm, no reboot | arm/disarm OK |
| 002 | 2026-10-11 | Shakedown, known-good module late-loaded (`default_entry` method) | **rig bug**: `boot-disarm` died (no `/run/lock` yet). `Requires=` kept the module unloaded; disarmed by hand. Fixed |
| 003 | 2026-10-11 | Shakedown retry (`default_entry` method) | pass: disarm → insmod → GPU bound → collect → back in ~35 s |
| 004 | 2026-10-11 | Shakedown, one-shot EFI var method | pass. Limine consumed `LoaderEntryOneShot` itself |
| 005 | 2026-10-11 | Gate: oops | back unattended; **records lost** (panic beat btrfs commit). Recovered by hand. Fixed with `sync` |
| 006 | 2026-10-11 | Gate: softlockup | as 005; `collect/` written zero-length (kept as evidence) |
| 017 | 2026-10-11 | **REVIEW.md "do this first"**: stock linux-t2 amdgpu (7003), EDID + DPCD capture, held for me | **favourable**: tile blocks on both EDIDs, both panel IDs `AE32`, no DSC, `0x205`=01 on both links. Me: two correct halves, smooth cursor across the seam. (Test verdict said "no EDID": sysfs size-0 bug, fixed; `collect2/` has them.) |
| 018 | 2026-10-11 | **D0b**: PoC `02906a9`, `tiled_stitch=0` | **pass**: quirk attached; wired root link[0] ↔ slave link[1]; root re-read as `AE32` with tile block (attempt 1); DRM `TILE` set on both (group 1, 2×1); `0x205`=01 both; probe-defer rescued a first-probe SMU timeout; no new warnings. Me: A, two correct halves |
| 019 | 2026-10-11 | **D1 first boot**: PoC + `0024` (B2), defaults (`tiled_stitch=-1 tiled_genlock=0`) | **stitch engaged**: one connector eDP-1 5120×2880, peer stream on link[1], DP-1 non-desktop, tile phase 0 lines, link-health PASS after 1 second-tile recovery; root OTG1, peer OTG0 (L7's order). Me: A, one desktop, halves indistinguishable, cursor and windows cross the seam cleanly |
| 020 | 2026-10-11 | D1 campaign, first try | pass on re-judge: `stitch-check` matched `DP-1 (` inside `eDP-1 (ID 0)` (judge bug, fixed) |
| 021–040 | 2026-10-11 | **D1 gate**: 20 consecutive dev boots, `stitch-check`, `--after reboot` | **20/20 pass**. Phase 0 lines on every pass; 0 re-aligns; no new warnings; Hyprland one 5120×2880 monitor. 19/20 needed 1–2 second-tile recoveries: during Hyprland's startup modesets link[1] drops to CR-only (`0x202–0x204` = `11 11 80`), link-health retrains it within ~250 ms |
| 041 | 2026-10-11 | **D2 interactive**, me at the screen: scale 2, `mpv` test pattern across the seam, DPMS ×5, window drag / cursor parked on the seam | scale 2, video, seam: all A (clean). **DPMS: REVIEW.md L7 confirmed** on 5/5 wakes: `DP link(1) sink AUX not ready for training`, `enabling link 1 failed` at +0.5 s, link-health recovery +1.2 s, phase 0 +1.5 s. Visible as the left tile stretched over the whole panel for ~1 s, then a fade to the right picture (the root EDID's tile block: "if it is the only tile: scaled to fit"). I also see a ~10-frame after-image on fast motion: same on both halves, and same on known-good 4K → panel baseline, not the stitch. No screen locker on this machine, so no hyprlock test |
| 042–044 | 2026-10-11 | L7 regression: PoC + `0024` + `0025` (root stream first), `stitch-check` ×3 (I asked for 3 now, full 20 later) | **3/3 pass**; now root OTG0 / peer OTG1; startup second-tile recoveries 0 (were 1–2 on 19/20 boots) |
| 045 | 2026-10-11 | **D2 DPMS re-test** with `0025`, me at the screen | **L7 fixed**: 5/5 wakes with no `AUX not ready`, no link-1 failure, 0 recoveries, halves in phase at +0.6–0.8 s (was ~1.5 s). Me: picture straight back, no stretched left half |
| 046–048 | 2026-10-11 | 20-boot regression of `0025` (paused after 3 to get to the resolution work) | 3/3 pass, 0 recoveries |
| 049 | 2026-10-11 | Scaled modes `0026`, first build | **fail**: all 20 scaled modes pruned in `mode_valid` ("No DP link bandwidth"): the native crtc timing was not copied during the probe (crtc fields are zero until the probe ends). Fixed, `0026` amended |
| 050 | 2026-10-11 | Scaled modes `0026` (amended), held, me at the screen | **pass**: 21 modes (native + 20 scaled). Hyprland set every one: peer attached, phase 0, no recoveries, no errors (`modesweep.txt`). Me: 2560×1440, 1024×768, 640×480 all clean (seam invisible, 4:3 bars equal, cursor fine) |
| 051–057 | 2026-10-11 | 20-boot regression, `stitch-full` (boot checks + DPMS ×3 + 2560×1440 round trip, from logs), PoC + `0024`–`0026`, me away | 6/7 boots pass the boot checks. Wake: `enabling link 1 failed` on 2 of 21 wakes (053, 056; L7 residue, recovered by the next commit). **057 crashed**: link-health FAILed on both links at boot, its escalation **self-deadlocked** (`amdgpu_dm_tiled_request_full_modeset()` locked `connection_mutex` around `drm_connector_set_link_status_property()`, which locks it again); hung-task panic after 122 s, back on known-good unattended. Netconsole caught the stacks (`057/netconsole-hung-tasks.txt.zst`). Fixed as series `0027` |
| 058–077 | 2026-10-11 | **20-boot regression, `stitch-full`**, PoC + `0024`–`0027`, me away | **20/20 boots pass**, 0 crashes, 0 startup recoveries; 40/40 scaled-mode round trips clean; 57/60 DPMS wakes clean. 3 wakes (072, 074, 077) with the residual link-1 failure: with DP debug on (072) link 1 trains fully (CR, EQ, aligned) but the sink drops interlane alignment ~6 ms after the training pattern ends (`0x204=80`, `0x205=00`, DC "Link loss"), on all 4 attempts over ~0.8 s; the next commit succeeds. Self-healing, cosmetic (brief stretched left tile). Open |
| 078 | 2026-10-11 | **D3 soak-lite**, 2 h held, PoC + `0024`–`0027`, exercise every 10 min (DPMS + 2560×1440 round trip) | **clean**: 0 warnings, 0 link-health failures, 0 phase errors, 1 healed link-1 wake; idle memory clock 98 MHz every sample (= 4K driver), GPU 9–10 W idle (4K: 9 W) |
| 079 | 2026-10-12 | **t2linux-based build** (`series-t2linux/`), `stitch-full`, then Hyprland scale 2 + 5 suspends (`rtcwake -m mem`, deep S3) | pass; **5/5 suspends** (45–180 s asleep): resume ~3.5 s, peer re-added, phase 0, link-health PASS with 0 recoveries; one-time `kfd_device.c:1139` WARNING on the first resume (KFD, not display) |
| 080–089 | 2026-10-12 | t2linux-based build, 10× `stitch-full` | **10/10 pass**, 30/30 wakes clean, 20/20 mode round trips |
| — | 2026-10-12 | Installed as the default (`updates/`), rig per-boot units off, scale 2 | 3/3 plain reboots (deferral fired and rescued each time, stitch up, no warnings); 3/3 suspends on the real default |
| — | 2026-10-12 | Default boot, ambient-style checks before handing over: 100 DPMS cycles, 10 more suspends | DPMS: 96/100 clean, 4 with the residual link-1 retrain, 0 link-health failures, 0 warnings, phase 0 throughout. **Suspends 10/10** (13/13 on the default in total) |
| 090 | 2026-10-12 | Candidate: `0001`–`0035` + a first wake fix (poll link 1's post-training status for 100 ms), `stitch-full`, held. Then 20 DPMS; `tiled_fault` 2 / 5 / 9; a DPMS cycle in fallback | `stitch-full` pass. Poll refuted: 27 of 27 alignment blips over 20 wakes never cleared within 100 ms during the idle pattern, yet every wake was healthy at link-health pass 1 with 0 recoveries. `tiled_fault=2`: 2 recoveries, PASS. `=5`: FAIL → **in-kernel modeset 1/2** → PASS (L5 works). `=9`: fallback entered, but that first L6 design changed the mode list; Hyprland ignored it, and the next DPMS-on was rejected: **panel black** (me). Back to known-good by reboot. L6 redesigned |
| 091 | 2026-10-12 | Candidate `0001`–`0036` (L5, L6 squeeze, `tiled_fault`, skip link 1's post-training loss check), `bcea67d6…`, `stitch-full`, held. 100 DPMS; `tiled_fault=9`; DPMS and S3 in fallback | `stitch-full` pass. **100/100 wakes PASS with 0 recoveries**, 46 alignment blips logged and left to link-health, all healed (was 4/100 visible stretched wakes). L6: "driving the root tile alone, desktop squeezed into it"; me: **whole desktop, correct proportions**; stays so after a DPMS cycle and an S3 resume. No re-latch (no slave HPD while link 1 is undriven): sticky until reboot |
| 092–101 | 2026-10-12 | Candidate `bcea67d6…`, 10× `stitch-full` regression, me away | **10/10 pass**, phase 0, 0 recoveries, 30/30 wakes, 10/10 mode round trips |
| — | 2026-10-12 | Arm D via `gpu-boot-check --campaign` (not the rig): `series-t2linux/` + `0031` reorder + `0032` KFD fix, 30 warm boots, then 30 DPMS cycles + 3 suspends on the last boot | **30/30 first probes OK** (no deferral, no MODE1 reset); 33/33 link-health PASS, 0 FAIL, 1 link-1 retrain (residual glitch), 3/3 suspends, **no KFD WARNING**. Promoted to the default (`62832c21…`) |
| 007–016 | 2026-10-11 | **R1 gate**: oops, softlockup, hardlockup, hung ×2 each; deadline timeout; refused (tampered module) | **10/10 back on known-good unattended**, records intact, guard green after each |

Gate timings (arm → back on known-good): oops ~110 s, soft/hard lockup ~140 s, hung task
~340 s (detector period 120 s), deadline 3 min → ~265 s. Netconsole delivered nothing from
the crash runs (no carrier yet when the module crashed ~2 s after insmod); fixed after the
gate by waiting for carrier.
