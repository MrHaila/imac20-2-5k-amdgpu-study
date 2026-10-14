# Results

This document lists what was measured, how it was measured, and what was not measured.
Every number links to a run in [`../evidence/run-journal.md`](../evidence/run-journal.md).
The journal also lists the failures.

**Which build?** "Default" is patches `0001`–`0033`, the build I use every day. "Candidate" is
`0001`–`0037`. "Earlier build" is `0001`–`0031`. See the [README](../README.md#two-builds-default-and-candidate).

## Test machine

| Item | Value |
|---|---|
| Machine | iMac20,2 (27-inch, 2020) |
| GPU | Radeon Pro 5700 (Navi 10), display engine DCN 2.0 |
| OS | Omarchy (Arch Linux), Hyprland |
| Kernel | `linux-t2` 7.2.9 (`7.2.9-arch1-Watanare-T2-1-t2`) |
| Compiler | gcc 16.2.1 |
| Default build module | sha256 `62832c210db33aa0ed33da0efe7c9a36f7a9e7f29445af0d18bf65bf27b28e52` |
| Candidate build module | sha256 `bcea67d644db942d1e0db9ff0ef3a9237de47b23ccb91d9bfc146c49561e5611` |
| Earlier build module | sha256 `9a5bf7c7219f37176a511dcbd68a0e629c0bba1f2457ce2bfe40e670421d3384` |

## Method

There are two kinds of test.

**Rig runs.** An automated test rig (not published) boots a test module one time.

1. If the boot hangs, crashes, or runs too long, the next boot uses the known-good module.
2. After the boot, a script reads the kernel log and the display state. It decides pass or
   fail. The script does not look at the screen.
3. I looked at the screen only in the runs marked "me at the screen".

Because the rig loads the test module late (after the disk unlock), **rig runs cannot
measure the GPU start-up problem.** That needs normal boots.

**Normal boots.** The module is installed as the real driver and the machine reboots. These
runs measure start-up (the first probe) and daily use.

The rig itself was tested first. In 10 of 10 deliberately bad boots (oops, soft lockup, hard
lockup, hung task, deadline timeout, tampered module), the machine came back to the
known-good module without help (runs 007 to 016).

### What "pass" means for a rig boot

A boot passes when all of these are true:

- The kernel log has no new `WARN`, `BUG`, `Oops` or `*ERROR*` line.
- Hyprland shows one monitor of 5120×2880.
- The second tile is attached to the same CRTC (log lines `TILED_STITCH:` and `APPLE5K:`).
- The two tiles are in phase: the log reports a difference of 0 lines (the limit is 3).
- The link-health check passes.

## The GPU start-up problem (patches 0004, 0032, 0033)

On this iMac, `amdgpu` fails to start the GPU on about half of all boots. The error is
`SMU: No response msg_reg: 6` and `-62`.

All data: [`../evidence/reorder-campaign-2026-10-12.tsv`](../evidence/reorder-campaign-2026-10-12.tsv).
Method: warm reboots, 2026-10-12, `linux-t2` 7.2.9.

| Arm | Driver | Boots | First probe OK | 95 % interval |
|---|---|---|---|---|
| A | Vanilla 7.2.9 | 20 | 11 | 32–77 % |
| B | Vanilla + the reorder (`0032`) only | 100 | **100** | 96.4–100 % |
| C | B + my probe deferral (`0004`) | 20 | 20. The deferral never fired | 83–100 % |
| D | **Default build** (reorder + stitch + KFD fix) | 30 | **30**. No deferral, no mode-1 reset | 88–100 % |

- Every B, C and D boot logged Display Core before "SMU is initialized". Every A boot
  logged the opposite order. The mechanism changed, not only the rate.
- Over 150 earlier boots on 7.1 and 7.2, the vanilla first-probe success rate was 42–57 %.
- **Independent data:** `edschofield` on an iMac20,1 (Radeon Pro 5300, Navi 14): 13 of 20
  boots failed with the upstream order. 0 of 40 failed with the 7003 order
  ([comment](https://github.com/t2linux/wiki/issues/743#issuecomment-6163146905)).
- **KFD warning:** the one-time `kfd_device.c:1139` warning at the first resume came from
  any failed first probe. It is gone in arm C and D (no warning in 3 suspends on each).
  It is explained and fixed by `0033`.
- **Probe deferral (`0004`)** was tested on its own before the reorder was known, on mainline
  `linux` 7.2.3: campaign 8 (an earlier version) 35 of 35 boots OK, 20 of 20 failed
  probes rescued. Campaign 9 (version 4, the same added lines as `0004`) 30 of 30 OK, 14 of 14
  rescued. It works. The reorder makes it unnecessary here.

## Boot and display, by build

### Default build (`0001`–`0033`)

| Check | Result | Run |
|---|---|---|
| Normal warm reboots | 30 of 30 reach the desktop with one 5120×2880 monitor | arm D |
| Link health | 33 of 33 PASS, 0 FAIL, 1 retrain of link 1 | arm D |
| Display off/on cycles | 30 cycles. 1 needed a retrain (the short artefact) | arm D |
| Suspend and resume | 3 of 3. No KFD warning | arm D |

### Earlier build (`0001`–`0031`)

This build has the same stitch code. Most of the evidence comes from it.

| Check | Result | Runs |
|---|---|---|
| First stitched boot | Stitch active. Me: one desktop, no visible difference between the halves | 019 |
| 20 boots (an older build, before the wake fixes) | 20 of 20 pass | 021 to 040 |
| 20 boots | 20 of 20 pass, 0 crashes | 058 to 077 |
| 10 boots (rig) | 10 of 10 pass. 30 of 30 wakes clean. 20 of 20 mode round trips | 080 to 089 |
| Plain reboots, installed as the driver | 3 of 3. The deferral fired and rescued each time | — |
| 100 display off/on cycles on the installed driver | 96 of 100 clean. 4 had the short artefact. 0 link-health failures | — |
| Suspend and resume | 5 of 5 on a test boot (run 079). 13 of 13 on the installed driver. Resume takes about 3.5 s | 079 |
| 2-hour soak, with a display cycle and a mode change every 10 minutes | 0 warnings, 0 link-health failures, 0 phase errors | 078 |
| Idle memory clock and power | 98 MHz in every sample, same as the 4K driver. 9–10 W idle (4K: 9 W) | 078 |

One campaign (runs 051 to 057) was stopped after a **crash** in run 057. The link-recovery
code locked a mutex twice. A hung-task panic rebooted the machine into the known-good
module after 122 seconds. This was fixed (patch `0031`). It is listed here because the
counts above do not show it.

### Candidate build (`0001`–`0037`)

| Check | Result | Run |
|---|---|---|
| 10 boots (rig) | 10 of 10 pass. Phase 0. 0 recoveries. 30 of 30 wakes. 10 of 10 mode round trips | 092 to 101 |
| 100 display off/on cycles, me at the screen | 100 of 100 pass with 0 recoveries. 46 alignment blips were logged and healed by themselves. I saw none of the short artefact | 091 |
| Fallback (`0035`), with a **simulated** failure | The desktop appeared whole with the right proportions. It stayed so after a display cycle and an S3 resume | 091 |
| In-kernel recovery (`0034`), with a **simulated** failure | With the test switch at 5, link health failed once. The kernel's own modeset (the first of the two it may try) brought the second tile back. This ran on a slightly earlier candidate (`0001`–`0036` plus a first version of the wake fix) | 090 |

What the simulated tests mean: the switch `amdgpu.tiled_fault` (patch `0036`) makes the
driver report the second tile as not ready. It does not break the hardware. The real
failure has never happened on my machine. These tests show that the recovery code runs
and that Hyprland accepts the result. They do not show that the recovery works for a real
failure.

The first design of the fallback (run 090) changed the mode list. Hyprland ignored that, and
the next display-on was rejected: the panel stayed **black**. The machine went back to the
known-good module by reboot. I redesigned the fallback to keep the mode list unchanged (run 091).

### Resolutions

The driver offers 21 modes: the native 5120×2880 mode and 20 scaled modes. These include
the macOS set, and common 16:10 and 4:3 modes. The GPU scales them. The panel timing does
not change.

- Hyprland set all 21 modes, one after the other. The second tile stayed attached and no
  error appeared (run 050).
- Mode round trips (2560×1440 and back): 40 of 40 (runs 058 to 077), 20 of 20
  (runs 080 to 089) and 10 of 10 on the candidate (runs 092 to 101).
- I looked at 2560×1440, 1024×768 and 640×480. The seam was not visible, the bars of the
  4:3 mode were equal, and the cursor behaved.

### Code size

Against Arch tag `v7.2.9-arch1`, the 37 patches add about 4,000 lines. About 3,100 of
the added lines come from the community patches without change. The default build
(`0001`–`0033`) compiles with gcc 16.2.1 without warnings.

## What was not measured

Please read this list. It is as important as the results.

- **Visual correctness by instrument.** There was no camera and no frame-CRC check.
  Every visual verdict is my eyes.
- **A second machine running my patches.** No other iMac has run them. `edschofield`'s
  iMac20,1 data above is about patch `7003`, not about this series.
- **The candidate build in daily use.** It has 10 rig boots and 130 wakes. It has not had a
  normal reboot, and only one suspend (in fallback mode).
- **A real second-tile failure.** I have never seen link 1 fail for good. Items L5 and L6
  were found by code review, and fixed and tested with simulated failures.
- **A 24-hour soak.** The longest soak was 2 hours, on an earlier build.
- **A 30-boot campaign on the candidate.** The default has 30 normal boots.
- **GPU reset recovery.** It was not tested.
- **A screen locker.** None is configured on the test machine.
- **Rendering performance.** No benchmark was run.
- **`amdgpu.tiled_genlock=1`.** It was never run on hardware. All results use the default (0).
- **Other compositors.** Only Hyprland.
- **Kernel 7.2.10.** The series applies with patch `0003` dropped. I did not build or boot it.
- **The user-space rescue script.** My machine has a script that rebinds `amdgpu` after
  a failed first probe. I did not record whether it ran in each earlier boot. In arm D
  the first probe never failed, so it was not needed.

## Reproduce the numbers

The raw logs stay on my machine. The run journal lists the evidence that each run
produced. To get your own numbers, follow [TESTING.md](TESTING.md). It lists the log lines
that show the same checks.
