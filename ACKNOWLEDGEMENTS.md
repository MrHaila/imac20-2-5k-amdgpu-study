# Acknowledgements

This work exists because other people did the hard parts first and shared them. This file lists who did what, as far as AI could trace it. If this is wrong or incomplete,
please open an issue and I'll correct it.

## People and projects

| Who | What they did | Where it is in this repository |
|---|---|---|
| **mforce2** | Found how to wake the second tile of the panel | Lean core (patch 0004) |
| **erik2** | Wrote the first single-display stitch | Stitch layer (patch 0005) |
| **taprobane99** | Ported the patches to kernel 7.2.x. Also [`iMac5KLinux`](https://github.com/taprobane99/iMac5KLinux) | Indirectly, through the patcher |
| **Ahmad Al-Awadi** ([@ahmadtv](https://github.com/ahmadtv)) | The original patcher, the first Omarchy backend, the 7.2 lean core and the GPU-reset patches in [`omarchy-imac18-3-patch`](https://github.com/ahmadtv/omarchy-imac18-3-patch) | Patches 0004 to 0005 |
| **Mark Pronkin** ([@MarkPronkin](https://github.com/MarkPronkin)) | Maintains [`imac5k-universal-linux-patcher`](https://github.com/MarkPronkin/imac5k-universal-linux-patcher). Wrote the link-recovery, resume and resync fixes, and the DCE genlock work | Patches 0004 to 0015 (commit `c1086d2b438c`) |
| **br1uhnz** | [`retina-5k-imac-linux`](https://github.com/BR1UHNz/retina-5k-imac-linux). Wrote the stitch gate, the plane checks, the link lifecycle (including the re-arming of `0x4F1` when the main panel powers up, which makes display wake and resume work) and the genlock fixes | Patches 0016 to 0020 (commit `6977710`) |
| **Atharva Tiwari** and **t2linux** (Aditya Garg and contributors) | [`linux-t2-patches`](https://github.com/t2linux/linux-t2-patches). The T2 kernel, and the first 5K patch for this model (`7003`, Atharva Tiwari). Atharva found that the display block must be set up before the power controller, which cures the start-up failure on this iMac too. Also [issue #33](https://github.com/t2linux/kernel/issues/33), which found a missing array terminator in `7003` | Patches 0001 and 0002 (commit `e167207`). Patch 0031 is the `7003` discovery hunk |
| **edschofield** | Tested patch `7003` on an iMac20,1 and posted traces in [t2linux/wiki#743](https://github.com/t2linux/wiki/issues/743). They confirmed the start-up result and the volatile `0x4F1` register on a second machine, and measured the link-training blip. Also the [Aquamarine patch](https://github.com/edschofield/aquamarine/tree/imac5k-development) that joins the tiles in Hyprland | No code of theirs is in this repository |
| **Alex Deucher** (AMD) | Replied to my bug report in [drm/amd#5726](https://gitlab.freedesktop.org/drm/amd/-/work_items/5726) with the reference commit `a5be7ad8f5f0` and a request for a patch. | Patch 0003 follows his direction. It does not use his suggested shape, for the reason in its commit message |
| **The AMD display and amdgpu authors** | The driver that all of this modifies | Everything |
| **Everyone in drm/amd#4455** | The long-running community thread | — |

The work in the first rows was passed from one project to the next. The patches in this
repository are the versions that Pronkin's and br1uhnz's repositories carry. Credit for them
is shared along that chain.

The patch numbers in this table are the numbers in `patches/`. The commit message of each
patch has an `[upstream: …]` line that names its exact source file and commit.

## What is new here

Patches 0003, 0021 to 0030 and 0032 to 0036, and the documentation. They were written with AI
assistance, and directed and tested by Teemu Haila.

## A note on credit

The git author of the community patches in this repository is the person who applied
them, because `git am` records that person. The real authors are named in the
`[upstream: …]` lines and in the table above. I did not change any of the community
code, except where a patch needed an offset to apply.
