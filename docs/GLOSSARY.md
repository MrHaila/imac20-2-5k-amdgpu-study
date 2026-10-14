# Glossary

The words that this repository uses. If a word is missing, please open an issue.

## The two builds

**Default build**
: Patches `0001`–`0033`. The build I use every day on my iMac. "Default" means "my machine
  boots it by default". It does not mean "official release".

**Candidate build**
: Patches `0001`–`0037`. The default build plus four patches that fix known problems. It
  passed my automated tests. I have not used it for daily work yet.

**Earlier build** or **earlier default**
: Patches `0001`–`0031`. My daily driver for part of 2026-10-12. Most of the run journal
  comes from this build or older ones.

## The panel

**Tile**
: One half of the panel. The panel is two tiles of 2560×2880 pixels. Each tile has its own
  DisplayPort link.

**Root tile** (main tile) and **second tile** (slave, peer)
: The root tile is the left half. It is on link 0 (the internal eDP link). The second tile
  is the right half, on link 1. The code uses the words *root*, *slave* and *peer*. They
  mean the same two things: the tile that is first, and the tile that follows.

**Stitch**
: Making the kernel show the two tiles as one 5120×2880 monitor. Without the stitch, the
  kernel shows two monitors. macOS does the equivalent in its display stack.

**`0x4F1`** (the dual-tile latch)
: An Apple-specific register in the panel (DPCD address `0x4F1`). Writing `1` wakes the
  second tile. It is volatile: it clears every time link 0 powers down (display off, or
  suspend). The driver has to write it again each time.

**Phase**, **genlock**
: Both tiles must start every frame at the same moment. *Phase* is the difference, in
  lines. *Genlock* is the method to align them. In the logs, "phase 0" is good.

**Link training**, **link health**
: *Link training* is how the GPU and the panel agree on a DisplayPort link setup. *Link
  health* is my name for the check that the driver does after each commit to see that both
  tiles still work.

## The GPU and the kernel

**amdgpu**, **DC** (Display Core), **DCN** and **DCE**
: `amdgpu` is the kernel driver. DC is the part of it that drives the screens. DCE is the
  older display engine (used by older iMacs). DCN is the newer one (Navi 10 and later,
  including this iMac). The community patches were written for DCE.

**SMU**
: The GPU's power controller. On this iMac it sometimes does not answer at start-up (error
  `-62`, `EnableAllSmuFeatures`). Patch `0032` fixes that.

**PSP**
: The GPU's security processor. It loads firmware. My patch `0004` was written because of how
  the PSP behaves after Apple's firmware.

**Probe**
: The moment the kernel starts a driver for a device. "First probe" is the first attempt
  in a boot. "Deferred probe" means the kernel tries again later.

**KFD**
: The compute part of the AMD driver (ROCm). Patch `0033` fixes a counter bug in its error path.

**DPMS**
: Display power management. "A DPMS cycle" is: display off, then on. Omarchy does this when the screen
  idles. It is the most frequent trigger of the short artefact.

**CRTC**, **connector**
: Kernel display objects. A CRTC creates the picture. A connector is a monitor output. The
  stitch makes **one** CRTC drive **two** DisplayPort links and show **one** connector.

## The platform

**T2**, **`linux-t2`**
: The Apple T2 security chip, and the Arch Linux kernel package with the patches that T2
  Macs need. Maintained by the [t2linux](https://t2linux.org) project.

**Omarchy**, **Hyprland**
: Omarchy is an Arch-based distribution. Hyprland is its Wayland compositor. Hyprland does
  not join tiled monitors by itself. GNOME's Mutter does.

**Aquamarine**
: The library under Hyprland that talks to the kernel's display interface. A patch by
  `edschofield` teaches it to join tiles in userspace. This is the alternative to the
  kernel stitch.

## My test method

**Rig**
: My private test setup. It boots a test module one time, and always falls back to a
  known-good module on the next boot, even after a crash. It is not published.

**Run**
: One boot of a test module through the rig. Numbered in
  [`../evidence/run-journal.md`](../evidence/run-journal.md).

**`stitch-full`**
: My automated test of one boot. It checks the log, then does three DPMS cycles and one
  resolution round trip. It judges from the kernel log. It does not look at the screen.

**Arm A, B, C, D**
: The groups of the GPU start-up test. A = vanilla driver. B = vanilla plus the reorder
  (`0032`). C = B plus my probe deferral (`0004`). D = the stitched default build.

**B2, L5, L6, L7**
: Labels from the independent review of my first design
  ([`background/independent-review.md`](background/independent-review.md)). B2 = the
  unlocked cursor (`0028`). L5 = recovery depends on the compositor (`0034`). L6 = no
  fallback if the second tile is lost (`0035`). L7 = wake-up ordering (`0029`).
