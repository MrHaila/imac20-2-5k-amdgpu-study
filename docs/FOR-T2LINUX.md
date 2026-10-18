# For t2linux: how this relates to patch 7003

This page is for people who work on `linux-t2` and the 5K patch `7003`. It answers a
request in [t2linux/wiki#743](https://github.com/t2linux/wiki/issues/743) for the "one CRTC,
single eDP-1" patch. It also lists what could be reused without the rest.

I am not a kernel developer. The code was written by AI. Please read it as a study, not as a
proposal. Everything here is open to correction.

## Where the one-CRTC patch is

It is patches `0004`–`0030` of this series (see [PATCH-MAP.md](PATCH-MAP.md), groups C, D and E).
That is the code of Mark Pronkin and br1uhnz, which I adapted to the DCN display engine of
the iMac20,2.

**It replaces the display part of `7003`. It is not a patch on top of it.** The two designs
are different:

| | `7003` | This series |
|---|---|---|
| What userspace sees | Two connectors (eDP-1 and DP-1), each 2560×2880, timing-locked | One connector, eDP-1, 5120×2880 |
| What needs to join the tiles | The compositor (GNOME does. Hyprland needs [`edschofield`'s Aquamarine patch](https://github.com/edschofield/aquamarine/tree/imac5k-development)) | The kernel |
| fbcon / console | Cloned onto both tiles | One screen |
| Size | Small | About 4,000 lines, most of them from other projects |
| Maintenance | Low | High. It touches the hottest parts of `amdgpu_dm.c`. See [KNOWN-ISSUES.md](KNOWN-ISSUES.md), item 6 |

To try it on top of `linux-t2`, you cannot just add it after `7003`. Build from Arch
`v7.2.11-arch1` and apply `patches/` instead. [TESTING.md](TESTING.md) has the commands.

## Which findings in the thread I can confirm on the iMac20,2

These are from my machine (Radeon Pro 5700, Navi 10). The per-boot data is in
[`../evidence/reorder-campaign-2026-10-12.tsv`](../evidence/reorder-campaign-2026-10-12.tsv).

1. **The display-before-SMU reorder cures the start-up failure.** This is the
   `amdgpu_discovery.c` hunk of `7003`, here as patch `0031`, with the `{ }` terminator
   added.

   | Driver | Boots | First probe OK |
   |---|---|---|
   | Vanilla | 20 | 11 |
   | Vanilla + reorder only | 100 | **100** |
   | Reorder + my probe deferral (`0003`) | 20 | 20. The deferral never fired |

   `edschofield` found the same on the iMac20,1 (Navi 14): 13 of 20 failures with the
   upstream order, none in 40 boots with the reorder. See
   [his comment](https://github.com/t2linux/wiki/issues/743#issuecomment-6163146905).
   On every reorder boot, the log shows Display Core before "SMU is initialized". On every
   vanilla boot, the order is the opposite.

2. **`0x4F1` is volatile.** It clears when link 0 goes to D3, on suspend and on every
   display off. `edschofield` measured this in 36 of 36 cases on the iMac20,1
   ([comment](https://github.com/t2linux/wiki/issues/743#issuecomment-6136239556)). It is the
   same on the iMac20,2.

   In this series the register is written again in `enable_link_dp()`, right after the eDP
   power-up and the HPD-ready wait, before any link trains. That is **br1uhnz's code**
   (patch `0019`, from `retina-5k-imac-linux` fix 16), not mine. Patch `0028` (mine) also
   adds the main stream before the second one, because with the second one first, every
   display wake logged `sink AUX not ready`.

   Result on my machine: every S3 resume I have run came back at full 5K without user
   action (16 of 16 when I wrote my wiki comment). I do not know the reason for sure. One
   guess: in this series, the second tile's stream is created in the same commit as the
   main one. So the driver does not depend on catching a hotplug event from link 1.
   `edschofield` found that, in `7003`, the 80 ms HPD from link 1 sometimes arrives while
   interrupts are still disabled. I have not tested this guess.

3. **The post-training alignment blip.** After link training on link 1, DPCD `0x204` reads
   `0x80` (lane alignment lost) about 5 ms later, although training succeeded. DC counts it as
   a link loss and retrains, up to four times. I see this on about half of all display wakes. On
   about 6 % of wakes all four retries show it, and then the left tile appears stretched
   for up to a second. In my tests the sink was aligned again about 400 ms after video
   started, with no retrain. `edschofield` measured 12 of 86 training sequences with
   this pattern. Patch `0036` skips the retrain for the second tile and leaves the
   check to the link-health pass. That gave 130 of 130 clean wakes. **I do not know why the
   panel does this.**

4. **A KFD counter bug in vanilla 7.2 (still in 7.2.11).** After any failed first probe,
   `kgd2kfd_device_exit()` leaves `kfd_locked` at −1 for the lifetime of the module. The next
   suspend then does not evict KFD processes, and the first resume prints
   `KFD suspend / resume ref. error`. Patch `0032` fixes it (8 lines). With the reorder, the
   first probe does not fail, so the bug does not show. It is still a bug in the error path.

## What you could take without the stitch

| Patch | Why it stands alone |
|---|---|
| `0031` | It is already the `7003` hunk. Only the `{ }` terminator is added, as in your revised `7003` (`fffa0cb6`) |
| `0032` | A generic KFD fix, not specific to Apple |
| The idea of `0036` | Do not retrain the second tile on the post-training `0x204` blip. It would need adapting to `7003`'s DP-1 connector. In a `drm.debug=0x100` log, look for "Link loss" lines on wake |

## What I am not asking

- I am not asking `linux-t2` to carry this series as-is.
- I do not expect support. Questions and bug reports are welcome.
- I have not tested on an iMac20,1. If you try it, please read [TESTING.md](TESTING.md)
  first.

Thank you for all the hard work on on the t2 kernel and I hope this helps!
