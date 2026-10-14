# Known issues and open questions

This list is as complete as I can make it. Items 1 to 3 were seen on hardware. The
others are known limits of the design or of the evidence.

Each item says whether the **default** build (patches `0001`–`0033`) or the **candidate**
build (`0001`–`0037`) has it. See the [README](../README.md#two-builds-default-and-candidate)
for what these words mean.

## Seen on hardware

### 1. A short artefact on about 1 display wake in 25

**Default: yes. Candidate: not seen in 130 wakes (patch `0037`).**

After a display wake (DPMS on), the left half of the panel is sometimes stretched over the
whole panel for less than one second. Then the picture is correct. Counts on the default
and earlier builds: 3 in 60 (runs 058 to 077), 4 in 100 (installed build).

What I know:

- Link 1 trains fully (clock recovery, equalisation, lane alignment). About 5 ms after the
  training pattern ends, the second tile reports that lane alignment is lost
  (`DPCD 0x204 = 0x80`). The driver counts this as a link loss and retrains, up to four
  times. About half of all wakes show this blip. On about 6 % all four retries show it.
- If you wait, the second tile is aligned again about 400 ms after video starts, with no
  retrain (27 of 27 cases).
- Another tester saw the same pattern on an iMac20,1 with patch `7003`: 12 of 86 training
  sequences ([comment](https://github.com/t2linux/wiki/issues/743#issuecomment-6136239556)).
- `0037` skips the retrain for the second tile and leaves the check to the link-health pass.

What I do not know: **why the panel drops the alignment** while the idle pattern runs. I
have not ruled out my code, and I have not ruled out the panel. `0037` hides the symptom. A
real link loss is still caught by the link-health check, but I have never seen one.

### 2. A kernel warning at the first resume

**Default: fixed (`0033`). Candidate: fixed.**

It was a warning from `kfd_device.c:1139` at the first resume of a boot. It was not caused
by the stitch. Any failed first probe left KFD's global counter at −1. The reorder (`0032`)
removes the failed probe, and `0033` fixes the counter in the error path. I saw no warning
in the arm D boots. The cause is in vanilla 7.2.9. See
[FOR-T2LINUX.md](FOR-T2LINUX.md), item 4.

### 3. The panel shows an after-image on fast motion

**Both.**

About 10 frames of after-image are visible on fast motion. They are the same on both halves.
They are also the same on the 4K driver. I think this is a property of the panel and not
of the patches.

## Design limits

### 4. If the second tile is lost for good

**Default: not handled. Candidate: handled, but only tested with simulated failures.**

*(Independent review items L5 and L6; see
[`background/independent-review.md`](background/independent-review.md).)*

I have never seen the second tile fail for good on real hardware. The review said it could.

- **Default:** the driver asks user space for a full mode set through the `link-status`
  property. KWin and Mutter handle it. I do not know of such handling in Hyprland
  (aquamarine or wlroots). So the right half of the desktop would stay dark until a
  display cycle or a reboot.
- **Candidate:** the kernel does the full mode set itself, up to two times (`0034`). If that
  fails, it drives only the left tile and squeezes the whole desktop into it. The panel
  stretches it back (`0035`). The picture then has half the horizontal resolution. The
  fallback stays until reboot. I tried to make it leave the fallback when the second tile
  comes back, but that needs a hotplug event from link 1, and link 1 is not driven in the
  fallback. In practice you reboot.
- The first design of the fallback changed the mode list. Hyprland ignored that, and the
  panel went black (run 090). The shipped design leaves the mode list alone.
- The switch `amdgpu.tiled_fault` (`0036`) simulates failures. It tests the code. It does not
  prove that the recovery works for a real failure.

### 5. No native 4K mode while the stitch is active

**Both.**

The driver offers 5120×2880 and 20 scaled modes. It does not offer the panel's own 4K
mode. I chose scaled modes because the 4K mode would need the two-tile latch to be dropped
and set again on every mode change.

### 6. One kernel version

**Both.**

The patches apply to Arch `v7.2.9-arch1`. They also apply to `v7.2.10-arch1` with patch
`0003` dropped, but I have not built or booted that. The review predicts that a move of the
connector code in 7.3 will break about a third of the changes in `amdgpu_dm.c`, and that
each later kernel needs a half day to a day of porting, plus new tests. This is the main
reason why the work is a study and not a product.

### 7. Not suitable for upstream

**Both.**

The stitch layer (about 2,300 of the 3,200 lines in the reviewed stack) makes the kernel do
what DRM leaves to user space: it creates a synthetic EDID, hides half of a real panel, and
splits a framebuffer across two streams. Pronkin's patch header says that upstream will not
take it. I agree. The "lean core" (about 700 lines, with the iMac20 gate) is a possible
upstream candidate. It needs its authors' sign-off and an AMD reviewer who accepts
panel-ID quirks. The better long-term path is tiled-output support in the compositor. For
Hyprland, [`edschofield`'s Aquamarine patch](https://github.com/edschofield/aquamarine/tree/imac5k-development)
already does it.

### 8. Per-frame genlock is untested

**Both.**

The module parameter `amdgpu.tiled_genlock=1` selects a per-frame reset of the two tile
timings. It was **never run on hardware**. One known gap is left in this mode: the reset
trigger is not removed when an output stops using it (item A6 in the notes). The default
(`0`) aligns the tiles once and re-aligns them only if they drift by more than 3 lines.
All results use the default.

## Limits of the evidence

### 9. One machine, one observer

See [RESULTS.md](RESULTS.md), "What was not measured". The same list applies here.

### 10. Resume on other machines

Another tester reports that resume with patch `7003` is flaky on an iMac20,1 (the display
comes back stretched, 2 of 3 resumes), and that a tested fix crashed
([thread](https://github.com/t2linux/wiki/issues/743)). My machine resumed correctly in
every S3 test (16 of 16 at the time of writing, and 3 of 3 on the default). This series
does not fix `7003`. It is a different design. I do not know if it would behave the same on
an iMac20,1.

### 11. Assumptions about other iMacs

- The gate accepts `iMac20,1` and `iMac20,2`. Only the 20,2 was tested.
- The pairing assumes that the internal eDP link is link 0 and the second tile is link 1.
  This is true on the test machine.
- The second tile is driven in a plain DisplayPort mode when its panel ID is `AE31` or
  `AE32`. The test machine reports `AE32`.
- The panel does not use DSC (compression). The code does not handle DSC.
- Display Core starts before the SMU after `0032`, so it cannot read the SMU's clock table
  and uses built-in defaults. On the iMac20,1 this falls back to Navi 14 defaults
  ([comment](https://github.com/t2linux/wiki/issues/743#issuecomment-6163146905)). I did not
  look at this on the Navi 10, and I have seen no problem. A maintainer may ask.

### 12. The module is unsigned and has no BTF

The module that I test is unsigned, so the kernel marks itself tainted. I remove the
BTF section because of a `pahole` version mismatch. This costs only BPF type information
for `amdgpu`.

## Process caveats

### 13. AI-written code, checked by me for behaviour only

AI wrote the new code (patches `0022` to `0037`, except the `0032` hunk), ported and applied
the community patches, wrote the documentation, and ran most of the tests through the rig.
Patch `0004` and the research behind it were also developed with AI. I directed the work,
checked the screen, and decided what to keep. A `Co-Authored-By: Claude` trailer in a patch
means that the AI wrote it or applied it. No kernel expert audited the code. The patches
have no `Signed-off-by` line, because they are not meant for submission.

### 14. The test rig is not published

The rig that boots test modules, enforces the known-good fallback, and judges the logs is
specific to one machine. It is not in this repository. The run journal lists what each run
checked. I will describe the rig if someone wants to build their own.

## Open questions

- Why does the second tile drop lane alignment after training on most wakes (item 1)?
- Does a second iMac20,x (20,1, or a different GPU) behave the same?
- Does anything other than Hyprland work with the stitched output, and does anything
  handle the `link-status` request (item 4)?
- Would tiled-output support in the compositor be easier than the kernel layer? For
  Hyprland, `edschofield` has already shown that it works.
- Should the probe deferral (`0004`), which I sent to amd-gfx on 2026-09-26, be withdrawn
  in favour of the reorder? I have not seen replies to it. I could not check the list archive
  while writing this.
