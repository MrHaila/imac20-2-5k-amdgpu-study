# Testing this driver on your machine

Thank you for testing. This page tells you how to build the driver, install it, check it,
go back, and report what you see.

## Which build? Default or candidate

You can build two versions. They differ by four patches.

| Build | Patches | Choose it if |
|---|---|---|
| **Default** | `0001`–`0033` | You want the build that I use every day. **Start here.** |
| **Candidate** | `0001`–`0037` | You want to help me check four fixes that I have not used for daily work yet. It also adds a switch to simulate failures. |

Details: [README](../README.md#two-builds-default-and-candidate). The difference in
behaviour is small in normal use. The default can show a short artefact on about 1 display
wake in 25. The candidate did not in 130 wakes.

If the candidate misbehaves, go back to the default. If the default misbehaves, go back to
your normal driver (see "Go back").

## Before you start

**This is an experimental kernel driver. If it fails, your internal display stays dark.**
Do not test on a machine that you need for work in the next hours.

You need all of these:

- An **iMac20,2**. An iMac20,1 matches the same gate, but nobody has tested it.
  Check with `cat /sys/class/dmi/id/product_name`.
- The kernel **`linux-t2` 7.2.9** (`uname -r` shows `7.2.9-arch1-Watanare-T2-1-t2`), with
  the matching `linux-t2-headers` package. The patches are made for this version. They
  also apply to 7.2.10 if you leave out patch `0003`, but I have not built or booted that.
  A module must match your running kernel exactly (Step 3).
- A way to reach the machine **without the display**. Use SSH from a second computer. Test
  the SSH login before you start.
- A way to go back. Use a Btrfs snapshot, or a copy of your current module (Step 1).
- The Arch tools for a kernel module build: `base-devel`, `git`, `zstd`.
- About 3 GB of free disk space and about 5 minutes.

If you use a bootloader other than Limine, replace the Limine commands with the commands
that rebuild your initramfs.

## Step 1: Save the current module

The new module goes in the `updates/` directory. It does not replace the stock module. The
stock module stays where it is. Save a copy of any module that already is in `updates/`.

```bash
KVER=$(uname -r)
ls /usr/lib/modules/$KVER/updates/ 2>/dev/null    # a module here means you already have an override
modinfo -n amdgpu                                  # shows which file the system loads
```

If `updates/amdgpu.ko.zst` exists, copy it to a safe place now.

## Step 2: Get the source and apply the patches

```bash
git clone --depth 1 --filter=blob:none --sparse --branch v7.2.9-arch1 \
    https://github.com/archlinux/linux.git linux-7.2.9
cd linux-7.2.9
git sparse-checkout set --no-cone '/drivers/gpu/drm/amd/' '/scripts/' '/MAINTAINERS'

# Default build: the first 33 patches
git am $(ls /path/to/this/repo/patches/*.patch | head -33)

# Candidate build instead: all 37 patches
# git am /path/to/this/repo/patches/*.patch
```

`git am` needs a Git identity. If it asks, run `git config user.name` and `git config user.email` first. Any value works. The identity only names the local commits.

This downloads only the `amdgpu` part of the kernel tree. All patches of your build must
apply without error. If one fails, stop and open an issue with the error text.

## Step 3: Build the module

You build only the `amdgpu` module, against the installed headers.

```bash
mkdir -p include/trace   # amdgpu_trace.h includes itself through this path
make -C /lib/modules/$(uname -r)/build M=$PWD/drivers/gpu/drm/amd/amdgpu \
    KCFLAGS="-I$PWD/include/trace -I$PWD/include" -j"$(nproc)" modules
```

This took about 90 seconds on my machine. A warning about `pahole version differs`
is normal.

Then prepare the module:

```bash
strip --strip-debug -o amdgpu.ko drivers/gpu/drm/amd/amdgpu/amdgpu.ko
objcopy --remove-section .BTF --remove-section .BTF.base amdgpu.ko
zstd -19 amdgpu.ko -o amdgpu.ko.zst
modinfo -F vermagic amdgpu.ko.zst
```

The `objcopy` line removes the type information (BTF). If your `pahole` version differs from
the one that built the kernel, the kernel can refuse to load a module that has BTF. A
module without BTF loads normally.

**Check the vermagic.** The output of the last command must be the same as `uname -r`
followed by ` SMP preempt mod_unload`. If it is not, do not install the module.

Your module is not byte-identical to mine, because compilers and tools differ. My default
build had sha256 `62832c210db33aa0ed33da0efe7c9a36f7a9e7f29445af0d18bf65bf27b28e52`, and my
candidate build `bcea67d644db942d1e0db9ff0ef3a9237de47b23ccb91d9bfc146c49561e5611`. You do
not need to match them.

## Step 4: Install

```bash
KVER=$(uname -r)
sudo install -o root -g root -m 644 amdgpu.ko.zst /usr/lib/modules/$KVER/updates/amdgpu.ko.zst
sudo depmod $KVER
modinfo -n amdgpu          # must print the path under updates/
sudo limine-mkinitcpio     # rebuilds the boot image; the amdgpu module is inside it
```

The initramfs must be rebuilt. On Omarchy, the `kms` hook puts `amdgpu` inside the boot
image. The old module stays in the image until you rebuild it. If `updates/` does not
exist, create it first: `sudo mkdir -p /usr/lib/modules/$KVER/updates`.

Then reboot.

## Step 5: Check that it works

After the reboot, the whole panel must show one desktop. Then run these checks.

**1. The kernel log shows that the stitch is active.**

```bash
journalctl -k -b | grep -E 'APPLE5K|TILED_STITCH'
```

Look for lines that say, in this order or close to it:

- `APPLE5K: iMac20 tiles: tiled_stitch=… tiled_genlock=0 (…)`
- `APPLE5K: wired tile pair: root link[0] <-> slave link[1]`
- `TILED_STITCH: exposed only stitched mode … (tile 2560x2880)`
- `TILED_STITCH: added peer slave-tile stream link[1] 2560x2880 …`
- `APPLE5K: tile phase genlock=one-shot root link[0] OTG…, peer link[1] OTG…: 0 lines` (3 or less is accepted)
- `APPLE5K: link-health PASS: both tiles healthy after … recovery attempts`

The exact text can differ a little. The prefixes `APPLE5K:` and `TILED_STITCH:` are stable.

If none of these lines exist, the gate did not match. Check the product name
(`cat /sys/class/dmi/id/product_name`) and that `modinfo -n amdgpu` shows your module.

**2. The system shows one monitor.**

```bash
hyprctl monitors | grep -E 'Monitor|[0-9]+x[0-9]+@'
```

You must see one monitor, `eDP-1`, at 5120x2880.

**3. No new warnings.**

```bash
journalctl -k -b -p warning
```

Compare this with the output on your normal driver. A `lib/vsprintf.c` warning is known
from the stock driver. A `kfd_device.c` warning after the first resume is known too (see
[KNOWN-ISSUES.md](KNOWN-ISSUES.md)). Any other new warning is interesting.

**4. Look at the screen.** Move a window across the middle of the panel. Park the cursor
on the middle. Play a video across it. The seam must not be visible. The two halves must
have the same brightness.

**5. Display wake.** Turn the display off and on: `hyprctl dispatch dpms off`, wait 5
seconds, then `hyprctl dispatch dpms on`. Do it 10 times. Your idle daemon does the same when the screen blanks. Count how often the left half
stretches over the whole panel for a moment. I see this in about 1 of 25 wakes.

**6. Suspend.** Suspend and resume 5 times. Check that both halves come back.

**7. Resolutions.** The driver offers 21 modes. Change the mode of `eDP-1` in your Hyprland
monitor configuration, or with `hyprctl`. Try 2560x1440, and then go back to 5120x2880.
With scale 2, the interface is exactly 2x.

## If the screen stays dark

1. Log in over SSH.
2. Look at the log: `journalctl -k -b | grep -iE 'amdgpu|APPLE5K|TILED_STITCH' | tail -80`.
3. Go back (next section) and reboot.

## Go back

```bash
KVER=$(uname -r)
sudo rm /usr/lib/modules/$KVER/updates/amdgpu.ko.zst   # or put your saved module back here
sudo depmod $KVER
sudo limine-mkinitcpio
sudo reboot
```

If you cannot log in, choose an older Btrfs snapshot in the Limine boot menu.

A kernel update also removes the module. The `updates/` directory belongs to one kernel
version. The new kernel loads the stock driver. The patches do not apply to the new
version without work.

## What to report

Please open an issue with the following. Failure reports are as valuable as success
reports.

```bash
cat /sys/class/dmi/id/product_name
uname -r
modinfo -n amdgpu
lspci -nn -d 1002:   # your GPU
journalctl -k -b | grep -E 'APPLE5K|TILED_STITCH|amdgpu' > amdgpu-log.txt
```

Also tell me:

- **Which build** you tested: default or candidate.
- Which checks in Step 5 passed or failed.
- How many boots, wakes and suspends you tried, and how many failed.
- Your compositor and its version.
- Anything that looked wrong on the screen, even for a short time.

Do not post the full log if it contains information that you want to keep private. The
log can include serial numbers of the panel and the machine.

## Parameters

The driver has two module parameters (three in the candidate). Set them on the kernel command line.

| Parameter | Values | Meaning |
|---|---|---|
| `amdgpu.tiled_stitch` | `-1` (default), `0`, `1` | `0` turns the stitch off. You then see two connectors, one for each half. |
| `amdgpu.tiled_genlock` | `0` (default), `1` | `1` resets the two tile timings on every frame. **I never tested `1` on hardware.** |

Setting `amdgpu.tiled_stitch=0` is a useful first test. It checks that the two tiles are
found, before the stitch layer runs.

### Candidate only: `amdgpu.tiled_fault`

Patch `0036` adds `amdgpu.tiled_fault`. It makes the driver **pretend** that the second tile
is not ready, so that the recovery code runs. It does not damage anything. You can change it
while the system runs.

```bash
echo 2 | sudo tee /sys/module/amdgpu/parameters/tiled_fault   # then do one DPMS off/on
```

| Value | What the driver does at the next commits |
|---|---|
| `2` | One retrain of the second tile |
| `3` to `8` | The kernel redoes the full mode set (up to twice) |
| `9` or more | The kernel gives up, drives only the left tile, and squeezes the desktop into it (the fallback). **This stays until you reboot.** |
| `-1` | As above, and the second stream is not even created |
| `0` | Off |

Test with a value of `9` only when you can reboot. The desktop is whole, with the right
proportions, but at half the horizontal resolution. Please tell me what you see.

