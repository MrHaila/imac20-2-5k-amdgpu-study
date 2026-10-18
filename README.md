# Native 5K on the iMac20,2 under Linux: a feasibility study

This repository holds a set of kernel patches. The patches make the `amdgpu` driver show
the internal 5K panel of an iMac20,2 as **one 5120×2880 display**.

> **AI Slop Ahead!**
>
> The new code here was written by AI. I've directed the
> work and tested it on hardware. The community patches were written by their respective authors.
>
> This is a **feasibility study and a discussion starter**. It is **not** a proposal for
> upstream in its current state.

**Status: work in progress (last update 2026-10-14).** I run the "default build" below on my
own iMac every day. This repository is a snapshot of that work, with the evidence, so that
other people can read it, test it, or take parts of it.

## tl;dr

- **The problem.** The 5K panel is two 2560×2880 halves, each on its own DisplayPort link.
  macOS joins them. On Linux you get two separate displays, or a 4K picture.
- **What this does.** The kernel drives both halves and shows Hyprland **one** 5120×2880
  monitor. No compositor patch is needed.
- **Status so far.** One iMac20,2 (Radeon Pro 5700), `linux-t2` 7.2.11, Hyprland (Omarchy) tested to work well enough for my daily use. Code is unaudited.


## Two builds: "default" and "candidate"

You will see these two words in all documents here. They are my own names, so here is what they mean.

| Name | Patches | In plain words |
|---|---|---|
| **Default build** | `0001`–`0032` | The build **I use every day** on my iMac. I installed it as my real graphics driver on 2026-10-12. "Default" means "the one my machine boots by default". It does **not** mean "official release". **If you only try one thing, try this.** |
| **Candidate build** | `0001`–`0036` | The default build **plus four more patches** (`0033`–`0036`). The four patches fix problems that I already knew about (see the table below). It passed my automated tests, but I have **not lived with it yet**, and one of its fixes has only been tested with simulated failures. I am waiting to see if it holds up before I make it my default. |

You choose the build when you apply the patches: the first 32 patches for the default build,
all 36 for the candidate. [docs/TESTING.md](docs/TESTING.md) shows the exact commands.

In short: **the default is the one I trust more. The candidate is the one I think is better.**

### What the four extra patches change

| Problem in the default build | Candidate |
|---|---|
| On about 1 display wake in 25, the left half shows stretched over the whole panel for under a second. It heals by itself. | Not seen in 130 wakes (patch `0036`) |
| If the second half of the panel were lost for good, the right half of the desktop would stay dark until reboot. I have **never** seen this on real hardware. It comes from a code review. | The kernel retries by itself (`0033`). If that fails, it squeezes the whole desktop onto the left half, and the panel stretches it back (`0034`). Tested only with **simulated** failures (`0035` adds the test switch) |

### Results in short

All numbers: one iMac20,2, `linux-t2` 7.2, Hyprland. The details, the method and what was
**not** measured are in [docs/RESULTS.md](docs/RESULTS.md).

| Check | Default build | Candidate build |
|---|---|---|
| Boots that reach one 5120×2880 desktop | 30 of 30 warm reboots | 10 of 10 (test rig) |
| GPU initialises on the first try (no retry needed) | 30 of 30 (vanilla driver: 11 of 20) | not measured |
| Display off/on cycles without the short artefact | 29 of 30 (1 needed a retrain) | 130 of 130 |
| Suspend and resume | 3 of 3 (earlier build of the default: 13 of 13) | 1 of 1, in fallback mode. **Not otherwise tested** |
| Mode switches (2560×1440 and back) | 20 of 20 (earlier build) | 10 of 10 |
| Longest soak | 2 hours (earlier build) | none |

"Earlier build" means the first stitched build, patches `0001`–`0030`. It has the same stitch code. It was my daily
driver for part of 2026-10-12, before the GPU fix `0031` was added.

## What problem this addresses

The 5K panel of the iMac20,2 is a *tiled* display. It has two halves of 2560×2880 pixels.
Each half is fed over its own DisplayPort link.

- macOS joins the two halves and shows one display.
- Linux (`linux-t2`, Arch/Omarchy) shows two displays on the latest t2 kernel, or a 4K picture on older ones.

The patches here make the kernel join the halves. Hyprland then sees one ordinary monitor without additional changes.

There is a second, separate problem in the same machine. On about half of all boots, `amdgpu`
fails to start the GPU, because Apple's firmware leaves the GPU in a state that the driver
does not expect. Patches `0031` and `0032` deal with it. See
[docs/FOR-T2LINUX.md](docs/FOR-T2LINUX.md).

## What this is based on

Almost all of the code was written by other people. This repository combines it and adapts
it to the iMac20,2 (Radeon Pro 5700, Navi 10, display engine DCN 2.0).

| Source | What I took | Licence |
|---|---|---|
| [`MarkPronkin/imac5k-universal-linux-patcher`](https://github.com/MarkPronkin/imac5k-universal-linux-patcher) (commit `c1086d2b438c`) | The "lean core" and the "stitch layer", and 10 more patches. This repository is a fork of Ahmad Al-Awadi's [`ahmadtv/omarchy-imac18-3-patch`](https://github.com/ahmadtv/omarchy-imac18-3-patch). Both build on community work in [drm/amd#4455](https://gitlab.freedesktop.org/drm/amd/-/issues/4455): mforce2 (second-tile wake), erik2 (stitch) and taprobane99 (7.2.2 port) | MIT |
| [`BR1UHNz/retina-5k-imac-linux`](https://github.com/BR1UHNz/retina-5k-imac-linux) (commit `6977710`) | Fixes on top of that stack: stitch gate, plane checks, link lifecycle (including the wake-up of the second tile after power-down), genlock | GPL-2.0 |
| [`t2linux/linux-t2-patches`](https://github.com/t2linux/linux-t2-patches) (commit `e167207`) | The amdgpu parts of two patches (`2009`, `6001`), which the shipped `linux-t2` module already contains. Also the first 5K support (`7003`, by Atharva Tiwari). It gives two connectors, it showed that the panel can be driven, and it found that the GPU must set up the display block **before** the power controller. Patch `0031` is that one hunk | GPL-2.0 |
| [`edschofield`](https://github.com/t2linux/wiki/issues/743) | Not code that I used. His iMac20,1 tests and traces in t2linux/wiki#743 confirmed my findings independently, and sharpened them. His [Aquamarine patch](https://github.com/edschofield/aquamarine/tree/imac5k-development) is the userspace alternative to my kernel approach | — |
| Alex Deucher (AMD) in [drm/amd#5726](https://gitlab.freedesktop.org/drm/amd/-/work_items/5726) | The direction for the probe fix, through his reference commit `a5be7ad8f5f0` | — |

My own contribution is small. See [ACKNOWLEDGEMENTS.md](ACKNOWLEDGEMENTS.md) for the full
list, and for the exact origin of each patch.

**About 3,100 of the roughly 4,000 added lines come from the sources above without change.**
Each patch states its origin in the commit message, in an `[upstream: …]` line. The
git author on the patches is the person who ran `git am`. It is not the original author.

## What is new here

[docs/PATCH-MAP.md](docs/PATCH-MAP.md) lists all 36 patches with origin, size and status. In groups:

| Patches | What | Origin |
|---|---|---|
| `0001`, `0002` | The amdgpu parts of two t2linux patches | t2linux |
| `0003` | Probe deferral: retry a failed first probe once. **Now a fallback that never fires** on this machine, because `0031` cures the cause. Sent to amd-gfx on 2026-09-26 | Mine |
| `0004`–`0020` | The community stitch stack, **unchanged** | Pronkin, br1uhnz |
| `0021`–`0026` | Wiring for this machine: a gate so that only iMac20,x runs the tile code, and the glue for the newer display engine (DCN) | Mine |
| `0027`–`0030` | Fixes that testing found: a cursor locking bug, a wake-up ordering bug, a lower-resolution mode set, a deadlock | Mine |
| `0031`, `0032` | The GPU start-up fix, and a small bug fix in the compute driver (KFD) | `0031`: Atharva Tiwari's hunk. `0032`: mine |
| `0033`–`0036` | **The candidate patches** | Mine |

## Limits of the evidence

- **One machine.** One iMac20,2 with a Radeon Pro 5700. Nobody else has run this.
- **One kernel line.** I run the default build on `linux-t2` 7.2.11. The measured results come from an earlier 7.2 point release with the same patches. On 7.2.11 the default has my daily boots behind it, not a new test campaign, and the candidate is built but not yet booted.
- **One compositor.** Hyprland.
- **Not tested yet:** iMac20,1, other GPUs, a 24-hour soak, GPU-reset recovery, a screen locker.

## Do you want to test it?

Thank you. Testing on a second machine is the most useful thing that can happen to this
work. Start with [docs/TESTING.md](docs/TESTING.md). It explains how to build the driver,
how to install it, how to go back, and what to report.

**Warning.** If the driver fails, you have no display. Before you start, make sure you can boot back to a known-good kernel.

## Do you want to read it?

Start here, in this order:

1. [docs/GLOSSARY.md](docs/GLOSSARY.md) if some of the words are new.
2. [docs/PATCH-MAP.md](docs/PATCH-MAP.md) to see what each patch does.
3. [docs/FOR-T2LINUX.md](docs/FOR-T2LINUX.md) if you work on 7003 or `linux-t2`.
4. [docs/KNOWN-ISSUES.md](docs/KNOWN-ISSUES.md) for what I know is wrong.

## Repository layout

| Path | Content |
|---|---|
| `patches/` | The 36 patches, in order. Apply them to Arch tag `v7.2.11-arch1` |
| `docs/TESTING.md` | Build, install, verify, roll back, report |
| `docs/RESULTS.md` | The measured results |
| `docs/KNOWN-ISSUES.md` | What is wrong or unknown |
| `docs/PATCH-MAP.md` | One line per patch |
| `docs/FOR-T2LINUX.md` | How this relates to patch 7003, and what could be reused |
| `docs/GLOSSARY.md` | The words used here |
| `docs/background/` | Working notes from the development: the design and an independent review. Not written in simplified English |
| `evidence/` | `run-journal.md` is the log of every test boot, including failures. `reorder-campaign-2026-10-12.tsv` is the per-boot data of the GPU start-up test |
| `ACKNOWLEDGEMENTS.md` | Who did what |
| `LICENSE.md`, `LICENSES/` | Licences |

## A note to the authors of the work used here

You did the hard part: the reverse engineering of the panel and the design of the stitch. Thank you.

- If you want a different credit line, a different name, or removal, open an issue.
- If you want any of my changes, take them. No need to ask.
- Pronkin's patch header says that upstream will not take the stitch layer. Arguably a compositor-side solution (tiled-output support in Hyprland) is the better long-term path.

## Licence

The patches carry the licence of the kernel files that they change: mostly MIT for
`amdgpu`, and GPL-2.0 for the rest. The documentation is CC BY 4.0. See
[LICENSE.md](LICENSE.md).
