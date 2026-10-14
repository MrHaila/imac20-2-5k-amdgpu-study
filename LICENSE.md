# Licence

This repository has two kinds of content. Each has its own licence.

## Patches (`patches/`)

A patch is a modification of a kernel source file. It carries the licence of that file.
For this repository this is:

- **MIT** for most of the `amdgpu` display code (the files that carry an `SPDX-License-Identifier: MIT` line).
- **GPL-2.0-only** (with the Linux syscall note) for the other kernel files.

The community patches are published by their authors under compatible terms:
Pronkin's repository under MIT, br1uhnz's under GPL-2.0, and t2linux's under GPL-2.0.
The full text of GPL-2.0-only is in `LICENSES/GPL-2.0-only.txt`. The text of the MIT
licence and the copyright lines are in the kernel source tree and in the repositories that
we name in [ACKNOWLEDGEMENTS.md](ACKNOWLEDGEMENTS.md).

The new patches (0004, 0022 to 0031) are offered under the same licence as the files that
they change.

## Documentation (everything else)

The text of `README.md`, `ACKNOWLEDGEMENTS.md`, `docs/` and `evidence/` is licensed under
**CC BY 4.0**. The full text is in `LICENSES/CC-BY-4.0.txt`.

## No warranty

All content is provided **as is**, without warranty of any kind. Kernel drivers can cause
loss of the display and of data. You use this work at your own risk.
