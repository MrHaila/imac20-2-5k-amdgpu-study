# Native 5K as one connector on iMac20,2 (DCN 2.0): R0 design

**Status: phase R0 (recon) of `docs/5k-stitch-plan.md` (private notes, not published),
written 2026-10-10. This is a reading and dry-run exercise. Nothing was built, installed
or booted.** The only tree changes were throwaway copies under `~/src/5k-ref/scratch-r0/`.

Short answer to the R0 gate question ("is there a sensible home on DCN for one CRTC
driving two streams?"): **yes.** The stitch layer turns out to have almost nothing to do
with DCE. It lives entirely in `amdgpu_dm` and the display-independent DC core. The whole
reference stack applies cleanly to our 7.2.9 + probe-defer tree. The DCN-specific work is
small (about 150 new lines) and is concentrated in one place: the tile **genlock** (making
both halves scan out in lockstep).

---

## 0. Vocabulary (for a reader new to DRM and AMD DC)

| Term | What it is | Rough analogy |
|---|---|---|
| **DRM/KMS** | The kernel's display API. Userspace (Hyprland) sees *connectors* (outputs), *CRTCs* (scan-out engines) and *planes* (layers), and changes them with *atomic commits* | CoreGraphics display config + IOSurface layers |
| **Atomic check / commit** | A commit is first *checked* (validated without touching hardware, often as `TEST_ONLY`), then *committed*. Any rule we add must live in the check, or userspace learns about it too late | A transaction's validate-then-apply |
| **amdgpu_dm** | AMD's glue between DRM and its own display library, DC. Every DRM object maps to DC objects here | — |
| **DC** (Display Core) | AMD's OS-independent display library, shared with Windows. Below it, per-generation hardware code | — |
| **DCE vs DCN** | The two display-engine generations. DCE (to 2017, e.g. Polaris iMac18,3, Tonga iMac15,1, Vega iMacPro). DCN (2017 onwards; Navi10 is **DCN 2.0**). DC's core and `amdgpu_dm` are shared. The hardware sequencer (`hwss`), timing generators and scaler code differ completely | Same API, different silicon drivers |
| **dc_stream_state** ("stream") | One video timing going out of one link. One stream needs one **OTG** | One physical output signal |
| **OTG / TG** | Output Timing Generator: the counter that produces hsync/vsync for a stream. DCN 2.0 on Navi10 has 6 | A display's pixel clock + raster counter |
| **dc_plane_state** ("plane") | A framebuffer region scanned into a stream: `src_rect` (which part of the buffer), `dst_rect` (where in the stream) | A CALayer with contentsRect |
| **pipe / HUBP / DPP / MPC** | On DCN, each plane on a stream occupies a *pipe*: HUBP fetches memory, DPP scales/colours, MPC blends. A large plane may be **pipe-split** across two pipes | — |
| **ODM combine / DSC** | Ways for DCN to handle very wide or very fast timings: split one stream over two OTG/OPP back ends, or compress it. **Neither is needed here** (each tile is only 483 MHz) | — |
| **DML** | Display Mode Library: AMD's bandwidth/watermark calculator. It decides whether a configuration fits | — |
| **Genlock / timing sync** | Making two OTGs scan line N at the same instant. DC has a *one-shot* alignment (`enable_timing_synchronization`) and a *per-frame* reset (`triggered_crtc_reset`, `enable_per_frame_crtc_position_reset`) | — |
| **DPCD** | The DisplayPort sink's register space, read and written over the AUX channel. `0x4F1` is an Apple vendor register: writing 1 on the eDP root wakes the second tile | — |
| **Tile / DisplayID tile block** | EDID metadata saying "I am tile (x,y) of an N×M monitor". DRM exposes it as the `TILE` connector property. Hyprland ignores it | — |

The panel: one 5120×2880 panel, fed as **two 2560×2880 tiles** over two internal DP
links: `eDP-1` (link 0, the *root*, left half) and `DP-1` (link 1, the *slave*, right
half). One 5K stream is impossible: 966.5 MHz is beyond one HBR2×4 link.

---

## 1. What the stitch layer does

Source: `imac5k-universal-linux-patcher/patches/imac5k-stitch-layer-7.x.patch`
("erik2's stitch", +1038 code / +276 comment lines, 10 files). It sits on top of the lean
core. File:function references below name functions in the patched `amdgpu_dm.c`
unless a file is given.

### 1.1 The object model

```
userspace sees:   eDP-1 connector ── CRTC ── primary plane (one 5120×2880 framebuffer)
                  DP-1 connector  (non_desktop=1, ignored by compositors)

kernel builds:    dm_crtc_state.stream       → root stream on link 0, 2560×2880 tile timing, OTG a
                  dm_crtc_state.stream_peer  → peer stream on link 1, cloned timing,      OTG b
                  dm_plane_state.dc_state      → plane on root stream, src_rect x=0..2559
                  dm_plane_state.dc_state_peer → plane on peer stream, src_rect x=2560..5119
                  (both planes point at the same buffer object)
```

So the plan's hypothesis (one CRTC and one connector, two `dc_stream`s, two plane
states with left/right source rectangles) is exactly what the reference does.

### 1.2 Piece by piece

| Concern | How the stitch does it | Where |
|---|---|---|
| **Gate** | Module parameter `amdgpu.tiled_stitch` (-1 auto = on for wired Apple pairs, 0 off). Uses `link->tiled_role` / `tiled_pair_apple`, set when the lean core wires the pair | `amdgpu_dm_link_tiled_stitch_enabled()`, `amdgpu_dm_helpers.c:dm_helpers_wire_tiled_peer()` |
| **Modes** | Removes every mode from the root connector. Adds one synthetic mode: the tile's native mode doubled horizontally (hdisplay, hsync, htotal and clock ×2; vertical unchanged). So the line time and refresh are identical to the tile's. Common modes are suppressed. `mode_valid` rejects anything else | `amdgpu_dm_replace_tiled_stitch_modes()`, `amdgpu_dm_connector_get_modes()`, `amdgpu_dm_connector_mode_valid()` hunk, `amdgpu_dm_connector_add_common_modes()` hunk |
| **EDID / TILE property** | Builds a synthetic 2-block EDID (base + DisplayID 2.0 with one 5120×2880 Type VII timing) for the root. Hides the root's `TILE` blob so tile-aware compositors don't try to stitch twice. Keeps the kernel-side tile fields | `amdgpu_dm_tiled_stitch_build_synthetic_edid()`, `…_apply_synthetic_edid()`, `…_hide_tile_property()`, `…_edid_connector_update()` |
| **Slave connector** | Marked `non_desktop`, so compositors never give it a CRTC | `amdgpu_dm_connector_get_modes()` hunk |
| **Root stream timing** | After the stream is built from the 5120 mode, halve the horizontal timing and pixel clock back to the tile timing *before* the stream enters the DC context. So DC/DML only ever see a 2560×2880 stream. Stream `src`/`dst` = 1:1 tile | `amdgpu_dm_tiled_stitch_apply_tile_timing()` (called from `create_stream_for_sink()`), `update_stream_scaling_settings()` hunk |
| **Peer stream** | In the atomic check's CRTC-enable pass: create a stream on `root_link->tiled_peer`'s sink and copy the root's timing, src/dst and colour space. It has no DRM connector (`dm_stream_context = NULL`). It is added to the same `dc_state` **before** the root stream, so both are committed by the same `dc_commit_streams()` | `create_tiled_peer_stream()`, `dm_update_crtc_state()` enable hunk |
| **Peer teardown** | Removed alongside the root on disable or modeset. A "logical modeset" shortcut keeps the live pair when only the DRM mode object changed. That shortcut had a `BUG_ON`, fixed by `5k-logical-modeset-guard.patch` | `dm_update_crtc_state()` disable hunk, `amdgpu_dm_tiled_stitch_has_live_root_stream()` |
| **Planes (check)** | When the primary plane is added on a stitched CRTC with a framebuffer ≥ 2 tiles wide: set the root plane rects to the left tile. Create a second `dc_plane_state` from the same DRM plane state, set it to the right tile (`src.x = tile_w`), and `dc_state_add_plane()` it to the peer stream | `dm_update_plane_state()` hunk, `tiled_stitch_set_plane_rects()` |
| **Planes (commit / flip)** | In `amdgpu_dm_commit_planes()`: split the recomputed scaling info into left/right halves. After the root's `update_planes_and_stream_adapter()`, issue a second one for the peer stream with the same `flip_addr`/`plane_info` (same buffer). It runs under the same `dc_lock` but is **a separate DC update**, so the two halves are not latched atomically | `amdgpu_dm_commit_planes()` hunks |
| **Colour** | Copies regamma, gamut and CSC from the root stream to the peer | `amdgpu_dm_color.c:amdgpu_dm_update_crtc_color_mgmt()` |
| **Cursor** | Mirrors the hardware cursor onto the peer stream, shifted left by `tile_w`. Clips at the seam by bumping `x_hotspot`. Hooked into every cursor-programming site | `amdgpu_dm_tiled_program_peer_cursor()`, callers in `amdgpu_dm_update_cursor()`, `amdgpu_dm_commit_planes()`, `amdgpu_dm_plane.c:amdgpu_dm_plane_handle_cursor_update()` |
| **State lifetime** | `stream_peer` / `dc_state_peer` are reference-counted in duplicate/destroy | `amdgpu_dm_crtc.c`, `amdgpu_dm_plane.c` |
| **Re-sync after commit** | A 250 ms delayed work re-runs the timing sync after any commit, because the in-commit one-shot alignment can fire before the re-trained slave is running | `amdgpu_dm_tiled_resync_work()`, `amdgpu_dm_queue_tiled_resync()` |
| **Boot** | A second `drm_client_dev_hotplug()` after fbdev setup, so the peer exists before Plymouth draws (otherwise the right half is dark for about a second at the LUKS prompt) | `amdgpu_drv.c:amdgpu_pci_probe()` hunk |
| **Hotplug** | No special handling in the base layer. The slave is non-desktop, and the lean core ignores the slave's self-inflicted HPD pulses. `retina 14-stitch-gate` adds re-attach on slave re-HPD and a "can we stitch *now*" predicate with a fallback | `retina fixes/14` |
| **Suspend/resume** | DM's cached-state resume. Two follow-ups fix it: release the cached peer (`5k-resume-drop-cached-peer`), and arm link health and resync after resume (`5k-resume-arm-link-health`) | `dm_resume()` hunk |

### 1.3 Engine-agnostic or DCE-specific?

Measured, not guessed: a diff of the full lean stack (core + stitch + the five promoted
follow-ups) against our tree touches **no DCE or DCN hardware file at all**. It touches
only `amdgpu_dm/*`, `dc/dc.h`, `dc/dc_types.h`, `dc/link/*` (the generic link layer) and
`amdgpu_drv.c`/`amdgpu.h`. The only engine-specific lines are two comments naming
`dce112_validate_bandwidth`.

The DCE dependencies are **indirect**. The lean core's genlock flags both tile streams for
DC's per-frame CRTC reset, then relies on whatever the hardware sequencer does with that
flag. On DCE that path was broken, and the DCE-only patches exist to fix it:
`dce110-genlock-master-from-pipe0`, `dce120-enable-crtc-reset`,
`dce12-multisync-master-first`, `retina 13-dce-genlock`, `retina 19-dce10-pplib-in-sync`.
**None of those apply to DCN**, and none are needed in their DCE form. Section 4 says what
replaces them.

---

## 2. What the stitch needs from the lean core, and what 7003 offers on DCN

| Stitch dependency | Lean core provides | 7003 provides (DCN) | Verdict for the port |
|---|---|---|---|
| `link->tiled_peer` (root ↔ slave wiring) | `dm_helpers_wire_tiled_peer()` at EDID parse, keyed on Apple panel IDs (incl. `APP AE31/AE32`; our root reads **AE31** today) | nothing (hard-codes `link_index == 1` inline) | **Missing in 7003.** Take lean. Pin the peer to `link_index 1` under a DMI gate (new) |
| `tiled_role`, `tiled_pair_apple` | added by the stitch layer itself in `dc.h` + the wiring helper | — | stitch |
| Tile geometry (`connector->has_tile`, `num_h_tile == 2`, `tile_h_size`) | Comes from the DisplayID tile block once the root EDID flips. `amdgpu_dm_reprobe_tiled_root_after_slave()` re-reads it | 7003 writes `0x4F1` in the eDP branch of `link_detection.c:detect_link_and_local_sink()` **before** the EDID read, so the root probably reads its tile EDID first time | **Unknown on iMac20**: does the flipped EDID carry a tile block? (U1) |
| Second tile woken (`0x4F1` on root) | At detect (`apple_tiled_root_wake()`), at stream enable (`dp_write_tiled_stream_enable_latch()`), before link training (`link_apple_5k_slave_aux_ready()`) | At eDP detect, and at commit when eDP mode is 2560×2880 (never true once stitched) | lean covers more paths. 7003's evidence: on iMac20 one write at detect is enough for link 1 to detect normally |
| Slave detect with HPD low | `apple_tiled_slave_present()` (AUX probe) | nothing; on iMac20 HPD evidently rises by itself | lean's path is dormant if HPD rises. Harmless |
| Link-1 DP stream | The stitch creates it (`create_tiled_peer_stream()`) | Userspace creates it on `DP-1` (two connectors) | stitch |
| Genlock flagging | Flags **both** tile streams before `set_master_stream()` (`dm_enable_per_frame_crtc_master_sync()` hunk) | Flags **only** link 1. That stream then becomes the master/event source, and `dc.c:enable_timing_multisync()` skips event sources, so **nothing is armed**. 7003's per-frame genlock is a no-op | lean |
| HW per-frame reset source | DCE fixes (inapplicable) | `dcn10_hwseq.c:dcn10_enable_per_frame_crtc_position_reset()` picks the source OTG from `grouped_pipes[0]` only when that pipe is the event source. `enable_timing_multisync()` never passes the event source, so this is **dead code**; the source stays hard-wired to OTG 0 | **Missing.** Write new (§4.1) |
| One-shot alignment | Stock `program_timing_sync()`. Both tiles have identical timing, so they are "synchronizable" without help | Forces `resource_are_streams_timing_synchronizable()` to true for any eDP + link-1 pair, even with mismatched timings | Neither needed: the stitch clones the timing. Don't take 7003's override (it would "sync" a 4K root to a tile) |
| Slave panel mode | Forces `DP_PANEL_MODE_EDP` (alternate scrambler reset) on the slave (`link_edp_panel_control.c`). The iMac Pro needed this reverted by panel ID | Leaves link 1 in default DP mode, and that **locked fine** on 2026-10-10 | Add `0xAE31/0xAE32` to `dc_link_is_apple_tiled_dp_panel_mode()` so we match the known-good behaviour (new, 2 lines) |
| DMI gating | none (panel-ID gating only) | `iMac20,1`/`iMac20,2` in `amdgpu_dm_quirks.c` (correctly terminated) and in `amdgpu_discovery.c` (**missing `{ }`**, t2linux/kernel#33) | Add a DMI gate in front of the stitch (new). Copy the dm_quirks style, never the discovery table |
| Display IP init before SMU | — | `amdgpu_discovery_set_ip_blocks()` reorders DM before SMU on iMac20 | No reason found in the patch or the logs. It also changes what a failed probe has already done before probe-defer kicks in (§3.3). **Drop for the PoC.** Test it in D0 only if 5K fails without it (U7) |
| Firmware handoff at reboot | `amdgpu_pci_shutdown()` hunk: atomic shutdown, latch clears, root T12 | — | lean. Its value on iMac20 is unknown (U12) |

**Conclusion: 7003 is not a usable base for the stitch.** The stitch needs the lean core's
wiring and latch infrastructure, which 7003 lacks. Porting the stitch onto 7003 would
mean rewriting the lean core. 7003 contributes **evidence** instead: link 1 detects on
its own after one `0x4F1`, plain DP panel mode works, tile timing is 2720×~2962 @ 483.25
MHz on both OTGs (`set_pixel_clock_v7 … 4832500, tg = 0/1`), and DML validates the
two-stream configuration.

---

## 3. Does it apply? (dry-run results on our tree)

Method: `git archive HEAD drivers/gpu/drm/amd include/drm` from
`patches/amdgpu-imac20-2-vpost/build/linux-7.2.9` (v7.2.9-arch1 + probe-defer `b8d020e`)
into `~/src/5k-ref/scratch-r0/base`. Each stack was applied with `patch -p1 --forward` to
its own copy (`v-t2`, `v-lean`, `v-t2lean`, `v-retina`). The original tree and its git
state were not touched.

| Stack (in order) | Result |
|---|---|
| 7003 alone | clean (offsets only) |
| **Lean pair**: lean-core, stitch-layer, going-down-stop-resync, post-commit-link-recovery, logical-modeset-guard, resume-drop-cached-peer, resume-arm-link-health | **clean.** One hunk at fuzz 2: `link_dpms.c` disable-latch call, checked by eye, landed after `blank_stream()` as intended |
| 7003 then lean-core then stitch | applies. `dm_enable_per_frame_crtc_master_sync()` at fuzz 1, and the result is coherent: both flag loops end up present. Semantically redundant (double `0x4F1` writes, two flagging loops) |
| **Full retina stack** (Pronkin lean pair + 5 community follow-ups + 4 DCE/iMacPro patches + retina 13–20) | **clean** (one fuzz 1 in `16-link-lifecycle`, `link_dpms.c`) |
| **Recommended base** (§5.1: the above minus retina 17/18/19) | **clean**, same two fuzzed hunks |

Probe-defer touches only `amdgpu/amdgpu_device.c:amdgpu_device_init()`. No 5K patch
touches that file, so there is **no textual or functional overlap**. One interaction is
real, though. 7003's IP reorder would make the display block init *before* the SMU. Today
a probe that fails in SMU hw_init has not touched the panel yet. With the reorder, it
would already have woken the second tile and trained both links, and would unwind DM
before the deferred re-probe. Dropping the reorder keeps the probe-defer path exactly as
validated.

Note on building: our module is built from kernel.org source plus probe-defer, **not**
from linux-t2's source, which already contains 7003. So the PoC tree must not contain
7003. If the work is ever rebased onto linux-t2's own source, 7003 has to be reverted
first (or the t2lean combination above re-checked).

Diff sizes measured against base:

| Stack | Files | Lines added |
|---|---|---|
| 7003 | 8 | ~103 |
| Lean pair (7 patches) | 18 | ~1 970 (of which `amdgpu_dm.c` 1 507) |
| **Recommended base (§5.1, 17 patches)** | 23 | ~2 690 |
| Full retina stack (incl. 17–19) | 26 | ~2 750 (`amdgpu_dm.c` 1 995; DCE-only files ~130, inert on Navi10) |

---

## 4. DCN 2.0 specifics

### 4.1 Genlock: what actually happens on Navi10

DCE hook → DCN 2.0 counterpart (from `dc/hwss/dcn20/dcn20_init.c:47-49`):

| Role | DCE | DCN 2.0 |
|---|---|---|
| One-shot alignment | `dce110_enable_timing_synchronization` (GSL) | `dcn10_enable_timing_synchronization` (`dcn10_hwseq.c:2584`). Uses `optc1_enable_reset_trigger` on each slave with the master's OTG instance, waits once, then disables |
| Per-frame reset | `dce110_enable_per_frame_crtc_position_reset` (GSL; dead on DCE < 12, see retina UPSTREAM-BUGS #2) | `dcn10_enable_per_frame_crtc_position_reset` (`dcn10_hwseq.c:2668`). Calls `optc1_enable_crtc_reset(tg, source_tg_inst, …)` (`dcn10_optc.c:787`). This programs `OTG_TRIGA` with *source = vsync of OTG `source_tg_inst`* directly, so no GSL is involved. **This path actually works on DCN**, unlike DCE |
| Vblank-only align | — | `dcn10_enable_vblanks_synchronization` / `optc2_align_vblanks` (only for differing timings; not used here) |

Problems found on DCN 2.0:

1. **Hard-coded source OTG 0.** `dc.c:enable_timing_multisync()` (line 1612) passes only
   the non-master pipes. `dcn10_enable_per_frame_crtc_position_reset()` then arms each
   with source `0`. That is correct only if the master happens to sit on OTG 0. With the
   stitch, the peer is added to the context **before** the root, so it probably gets OTG 0
   and becomes master (`set_master_stream()` picks the first flagged stream at equal
   refresh). So it may work **by coincidence**, the same way it did on iMac15,1 ("DP-1 on
   OTG0, eDP-1 on OTG1"). Fix (new, ~40 lines, gated): put the master pipe first for the
   Apple pair (as Pronkin's `dce12-multisync-master-first` does for DCE 12), use
   `grouped_pipes[0]->stream_res.tg->inst` as the source, and arm/wait on entries
   `1..n` only. Never arm the master on its own vsync.
2. **Stale triggers.** Nothing on DCN calls `disable_reset_trigger` for a per-frame reset.
   An OTG that later drives a different display (for example a Thunderbolt monitor after
   the stitch falls back) would keep resetting on another OTG's vsync. Needs a teardown
   hook (new, ~25 lines) before per-frame reset is left on by default.
3. **Post-commit resync is a no-op on lit pipes.** `program_timing_sync()` only keeps
   unblanked slave pipes when `dc->config.use_pipe_ctx_sync_logic` is set. That flag is
   set for DCN 3.1+ only (`dcn31_resource.c:2130` and later), **not DCN 2.0**. So retina's
   "bug 4" applies to Navi10. The stitch's 250 ms resync re-aligns nothing once both tiles
   are lit; only the one-shot inside a modeset commit, while pipes are still blanked,
   aligns. Retina's `13-dce-genlock` has a generic **measured re-align**
   (`amdgpu_dm_tiled_phase_delta()` + `amdgpu_dm_tiled_measured_realign()`: read both
   OTG positions with `tg->funcs->get_position`, re-run
   `dc->hwss.enable_timing_synchronization` on the pair only if they are > 3 lines
   apart). It uses only `hwss`/`tg` function pointers that DCN 2.0 has. Today it is gated
   to DCE < 12. For DCN 2.0 we want it too.

Both tiles very likely run off the same DP reference clock: same 483.25 MHz request, and
iMac15,1 measured 0.0 ppm. So once aligned they should stay aligned, and the per-frame
reset may not be needed at all. **Recommended PoC genlock:** new parameter
`amdgpu.tiled_genlock`. `0` (default) = one-shot in the commit + measured re-align, no
per-frame reset. `1` = per-frame reset with the master/source fix from (1). D1 measures
the phase delta in both modes (`get_position` is readable from the realign code's log
line) and keeps the simpler one if it holds. Note the evidence conflict: Pronkin measured
"both flagged = locked, one flagged = sheared" on iMac18,3, while br1uhnz shows the DCE <
12 per-frame path can never arm. The difference Pronkin saw was probably the one-shot
master pick, not per-frame reset. Only hardware can settle it for DCN (U5).

### 4.2 Pipes, MPC split, ODM, DSC

- Navi10: 6 pipes / 6 OTGs (`dcn20_resource.c`). Two tile streams need 2 OTGs and 2 pipes,
  which leaves room for both Thunderbolt outputs.
- `pipe_split_policy = MPC_SPLIT_AVOID_MULT_DISP` (`dcn20_resource.c:739`). With two
  streams DC avoids pipe split unless DML requires it, and 483 MHz per pipe is far below
  Navi10's DPP clock limits. If a split ever happens, it is DC's normal per-plane split
  inside one stream. The stitch's two planes are already separate per stream, and DC
  handles each split on its own. `program_timing_sync()` skips split bottom pipes
  (`top_pipe`) and ODM slaves.
- **ODM/DSC not needed.** Each stream is 2560×2880 at 483.25 MHz, 10 bpc: about 14.5 Gbit/s
  on HBR2×4 (17.28). The 5120-wide mode never reaches DC as a stream, because
  `apply_tile_timing()` halves it inside `create_stream_for_sink()`, which `mode_valid`
  also uses. Failure mode to watch: if halving does not happen (the geometry predicate
  fails), DC sees a 966 MHz stream and rejects it. That is the "zero modes, black panel"
  case retina 14 fixes.
- **Shared framebuffer:** each HUBP fetches its own viewport of the same surface
  (`viewport.x = 0` / `2560`). 2560 is a multiple of every DCN 2.0 swizzle block width,
  so DCC/tiling alignment is fine.
- **Same line time by construction:** the 5120 mode has twice the htotal and twice the
  clock, so DRM's vblank timestamping (which uses the CRTC mode) and the OTG (tile timing)
  agree on line duration and refresh.

### 4.3 Cursor across the seam on DCN

DCN converts the cursor into plane space in `dcn10_set_cursor_position()`. With
`translate_by_source` it adds `src_rect.x`; amdgpu_dm sets that flag for every display
IP below 4.0.1, which includes DCN 2.0 (`amdgpu_dm_plane.c:1342`). Then
`hubp2_cursor_set_position()` (`dcn20_hubp.c:990`) subtracts `viewport.x` and disables
the cursor when it falls outside the viewport.

Worked through for the peer (src.x = viewport.x = 2560): the stitch passes
`x - 2560`, DCN adds 2560 and subtracts 2560, so the cursor lands correctly. When the
cursor straddles the seam, the stitch sends `x = 0, x_hotspot += overlap`, and hubp's
negative-offset logic clips it. On the root, a cursor in the right half is auto-disabled
("not visible beyond right edge"). **The DCE-written cursor math is correct on DCN 2.0
as-is.** An upstream comment there even mentions "tiled display that might enable the
cursor on both streams". Cursor size: DCN 2.0 `max_cursor_size = 256`
(`dcn20_resource.c:2480`). No DMUB cursor offload on Navi10 (no DMCUB).

### 4.4 The "Bandwidth violation" log line: benign

`CalculatePrefetchSchedule ***failed***. Bandwidth violation. Results are NOT valid` comes
from `dml/dcn20/display_mode_vba_20v2.c:2265`. It is printed **once per failed iteration**
of DML's VStartup search, which starts at 13 lines (`:2059`) and steps up one line at a
time (`:2366`) until the prefetch fits. It only prints at all because those boots ran with
`amdgpu.dyndbg=+p`.

Evidence that it is search noise, not a verdict:

- The plain **4K** boots log it too: 5 592 times (boots −3, 0) and 13 316 times (boot −2),
  against 24 577 times in the 7003 boot.
- `HTotal: 2720` is not tile-specific either: the 4K boots have it too, because the
  EDID's 2560×1440 DTD also has htotal 2720 (2560+48+32+80). The tile-specific marker is
  `vblank_start=2959` (2880 + 79).
- The 7003 boot went on to program both OTGs at 483.25 MHz and lit both halves.

The real worry is **memory-clock idling** (Pronkin/retina found the memory clock pinned
at max with two streams on DCE 10). On DCN 2.0, DML may switch memory clock in vblank only
if all streams are timing-synchronized (`dml/dcn20/dcn20_fpu.c:1343`). The stitch's
cloned timing satisfies the stock check, so idle memory clock is expected, but needs
measuring (U9).

### 4.5 Other DCN/iMac20 points

- **Flip atomicity:** root and peer are two `dc_update_planes_and_stream` calls. If the
  second lands after the next vblank, one frame shows a torn seam. Pronkin reports it
  seamless in practice on DCE. DCN has a real fix available: the GSL flip lock used for
  pipe-split immediate flips (`dcn20_setup_gsl_group_as_lock()`). That is D2 hardening,
  not PoC.
- **PSR:** if the root eDP advertises PSR and DM enables it, the root would self-refresh
  while the peer streams. Unknown whether this panel has PSR (DPCD `0x070`; iMac15,1's
  has none). Belt and braces for dev boots: `amdgpu.dcdebugmask=0x10` (DC_DISABLE_PSR),
  or skip PSR setup on a stitched root (5 lines).
  **Correction (PoC, 2026-10-10): PSR and Replay cannot engage on Navi10 at all.** In
  7.2.9, `amdgpu_dm_psr.c:amdgpu_dm_set_psr_caps()` returns early unless
  `dc->caps.dmub_caps.psr` is set, and only the DCN 3.0+ hardware sequencers set it
  (from DMUB firmware caps). `amdgpu_dm_set_replay_caps()` needs `dc->ctx->dmub_srv`.
  DCN 2.0 has no DMUB. So no gate is needed, and A7 is zero lines.
- **Link loss after enable is real here too:** the 7003 boot logged
  `Link(1) training attempt 1 of 4 failed … fail reason:(8)` (`LINK_TRAINING_LINK_LOSS`)
  at 17.7 s; the retry passed. That is the same family of problem that
  `5k-post-commit-link-recovery` handles on iMac18,3. Also
  `dp_verify_link_cap_with_retries: Link[1] … is_success=0` on the very first detection at
  11.7 s, then success later: the slave needs a moment after the latch.
- **No seamless boot on Navi10** (`amdgpu_device_seamless_boot_supported()` requires an
  APU), so firmware-OTG-binding surprises are out.

---

## 5. PoC port plan

### 5.1 Recommended base

**Pronkin's lean pair plus the stitch fixes, on our probe-defer tree, *without* 7003, plus
a small new iMac20/DCN adapter.**

Concretely, apply in this order (all verified to apply cleanly, §3). Take the files from
`imac5k-universal-linux-patcher/patches/` and `retina-5k-imac-linux/patches/`:

1. `imac5k-lean-core-7.2.x.patch`
2. `imac5k-stitch-layer-7.x.patch`
3. `5k-going-down-stop-resync.patch`
4. `5k-post-commit-link-recovery.patch` (taken because later patches anchor on it, and
   the 7003 boot already showed a link-1 link loss. Its recovery uses generic DC calls)
5. `5k-logical-modeset-guard.patch`
6. `5k-resume-drop-cached-peer.patch`
7. `5k-resume-arm-link-health.patch`
8. `imacpro-slave-dp-panel-mode.patch` (we extend its ID list, see A3)
9. `dce120-enable-crtc-reset.patch`, `dce12-multisync-master-first.patch`,
   `dce110-genlock-master-from-pipe0.patch` (DCE-only, inert on Navi10; kept only so later
   patches apply unchanged)
10. `5k-resync-postpone-on-modeset.patch`
11. retina `13-dce-genlock.patch` (we want its measured re-align, see A4)
12. retina `14-stitch-gate.patch` (never zero modes, retrying root EDID re-read, slave
    re-attach)
13. retina `15-stitch-planes.patch` (plane guard; **peer plane sent on every commit and its
    address refreshed in `prepare_fb`**. Without this, a `modetest` modeset with no
    following flip probably leaves the right half scanning a stale or zero address, which
    would wreck D1's first test)
14. retina `16-link-lifecycle.patch` (fixed pairing, idempotent latch, root-side wake on
    power-up, root-side going-down handoff)
15. retina `20-fixups-1.patch`

Skip: 7003 entirely; retina 17 (HPD-during-reset; revisit in D3), 18 (VCE, which Navi10
does not have), 19 (DCE 10 PowerPlay).

Why this instead of "7003 + port the stitch": §2. The stitch is built on the lean core's
data model. Every reference fix (Pronkin's crash fixes, retina's P0 findings) targets that
stack. And the combination has been validated on real hardware twice (iMac18,3 on DCE
11.2, iMac15,1 on DCE 10). We inherit the fixes for free and only own the DCN delta.

If a smaller first step is preferred, the minimum that is still testable is items 1–2 +
5 + 6 + the `prepare_fb`/every-commit part of 15 (hand-ported). That saves about 900 lines
but forks from the tested stack. Not recommended.

### 5.2 New code for DCN / iMac20 (the adapter)

| # | Item | Where | Size | Complexity / risk |
|---|---|---|---|---|
| A1 | DMI gate `iMac20,2` (+ `20,1`) in front of the stitch and tile wiring, with a **terminated** table | `amdgpu_dm_quirks.c` (`retrieve_dmi_info()` style), consumed in `amdgpu_dm_link_tiled_stitch_enabled()` | ~25 | low / low |
| A2 | Pin the pair: on a DMI-matched iMac20, wire only `links[0]` (eDP) ↔ `links[1]` (link_index 1, as 7003 does). Refuse any other DP link | `amdgpu_dm_helpers.c:dm_helpers_wire_tiled_peer()` (after retina 16) | ~15 | low / low |
| A3 | Slave in plain DP panel mode on iMac20 (matches 7003's working result): add `0xAE31`, `0xAE32` to `dc_link_is_apple_tiled_dp_panel_mode()` | `dc/dc.h` | ~3 | low / medium. If wrong, the right tile trains but shows nothing (`0x205 = 00`). Flip it and retest |
| A4 | Genlock mode for DCN 2.0: `amdgpu.tiled_genlock` (0 = one-shot + measured re-align, 1 = per-frame). Make retina 13's `amdgpu_dm_per_frame_crtc_reset_supported()` return the param on DCN 2.0, so mode 0 uses `amdgpu_dm_tiled_measured_realign()`, which calls `dcn10_enable_timing_synchronization` | `amdgpu_dm.c` | ~20 | low / medium (the one-shot jumps the slave OTG once when it fires on a lit pair; expect one glitched frame on the right half, only when drift > 3 lines) |
| A5 | Per-frame reset done right for mode 1: master first, source = master's OTG, arm/wait slaves only. Gated on `dc->config.apple_tiled_genlock` (new flag), so other DCN users are unaffected | `dc/core/dc.c:enable_timing_multisync()`, `dc/hwss/dcn10/dcn10_hwseq.c:dcn10_enable_per_frame_crtc_position_reset()`, `dc/dc.h` | ~40 | medium / medium (shared DCN code; a wrong source just fails to lock, `wait_for_reset_trigger_to_occur` times out and logs) |
| A6 | Per-frame teardown: `disable_reset_trigger` on the slave OTG when a stitched pair is disabled or falls back | `amdgpu_dm.c` stitch disable path, or `dcn20_hwseq` stream disable | ~25 | medium / low (D2; not needed while A4's default is 0) |
| A7 | No PSR/Replay on a stitched root | `amdgpu_dm_psr.c` / `amdgpu_dm_replay.c` gate, or just `amdgpu.dcdebugmask=0x10` on the dev entry | 0–5 | low / low. **Correction (PoC): 0 lines.** Both need DMUB, which Navi10 lacks (§4.5) |
| A8 | Rig-friendly logging: one `APPLE5K:` tag line per decision (pair wired, link indices, OTG instances of root/peer, which genlock mode, phase delta before/after). Most of it already exists in the reference stack | `amdgpu_dm.c` | ~30 | low / none |
| A9 | (D0 experiment only, default off) 7003's DM-before-SMU reorder behind a param, with terminated DMI table, in case 5K fails without it | `amdgpu_discovery.c` | ~25 | low code / **unknown effect**, and it interacts with probe-defer (§3) |

New code: **about 150 lines for the PoC** (A1–A5, A7, A8), plus about 50 more for A6/A9.
Reused verbatim: about 2 690 lines (§3).

### 5.3 What to skip for the PoC (D1) and add later

| Later phase | Item |
|---|---|
| D2 | A6 trigger teardown; GSL flip-lock for atomic seam updates (§4.5); confirm cursor across seam; Hyprland scale 2; DPMS (`hyprlock`) |
| D2 | Synthetic EDID: already in the stitch and kept, since removing it would fork. Hyprland reads modes from the kernel list, not the EDID, so it is not on the critical path |
| D3 | Suspend/resume (tie-in with `sleep-failure.md`); GPU reset (retina 16's reset arming, retina 17); warm-reboot firmware handoff on iMac20; memory-clock idle check; long soak |
| D3 | Probe-defer path with the stitch: a failed first probe now unwinds a DM that may have set the latch (only if A9 is on; without A9 the SMU fails before DM hw_init) |
| U | Fold A1–A5 into a form Pronkin's repo can take (it already lists iMac20,x as "untested"), and report the dead-code finding in 7003's dcn10 hunk and missing terminator to t2linux |

### 5.4 Suggested D0 change to the plan

The plan's D0 baseline is "probe-defer + 7003 (terminator-fixed)". Keep it as **D0a** (it
reproduces a known-good result and proves the rig sees both halves). Add **D0b**: the
recommended stack with `amdgpu.tiled_stitch=0`. That exercises the lean core's detection,
wiring and latch on iMac20 *without* the stitch, and answers U1–U3 and U6. It should give
two tiled connectors with a `TILE` property. Move to D1 (`tiled_stitch=1`) only after D0b
is clean.

---

## 6. Open unknowns that only hardware can answer

| # | Question | How to see it | Matters for |
|---|---|---|---|
| U1 | After `0x4F1`, does the root EDID flip (AE31 → AE32?) **with a DisplayID tile block** (2×1, 2560×2880), and does link 1's EDID carry tile (1,0)? The stitch geometry needs `num_h_tile == 2`. If there is no tile block, A1 must supply the geometry from the DMI match | D0b: `modetest -c` (`TILE` property), the EDID blobs in `/sys/class/drm/card1-*/edid`, `edid-decode` | everything |
| U2 | Is the slave panel ID in the lean list (AE31/AE32), so `dm_helpers_wire_tiled_peer()` fires? | D0b log: pair-wired line | A2 |
| U3 | Does link 1's HPD rise by itself (7003 evidence: yes), or does the AUX-probe path run? | D0b log | low |
| U4 | Is the latch volatile across DPMS off on this TCON (iMac15,1: yes; iMac18,3: no)? | DPMS off/on, read `0x4F1` via `/dev/drm_dp_aux*` (read-only) | D2 |
| U5 | Is one-shot alignment enough (stays within ≤ 3 lines), or does the TCON need per-frame reset? Does DCN per-frame reset (A5) lock cleanly? | A8 phase-delta logs in both genlock modes; camera on a seam-crossing diagonal | A4/A5 |
| U6 | Slave panel mode: does plain DP (A3) lock video (`0x205 = 01`)? | D0b/D1 DPCD read | A3 |
| U7 | Is 7003's DM-before-SMU reorder needed for anything? | Only if D0b fails to bring up link 1 | A9 |
| U8 | Does the root eDP have PSR? | DPCD `0x070`, debugfs `psr_capability` (root needed) | A7. **Moot on Navi10** (no DMUB, so DM never enables PSR; §4.5) |
| U9 | Does memory clock idle with the stitched pair (DML synchronized vblank)? | `pp_dpm_mclk` / `amdgpu_pm_info` at idle | D3 |
| U10 | Seam tear under motion from the two separate plane updates? | camera, video and window drags across the seam | D2 |
| U11 | Link-1 link loss after enable (seen once in the 7003 boot): how often, and does post-commit recovery fix it? | `APPLE5K: link-health` lines over the 20-boot D1 gate | D1/D3 |
| U12 | Warm reboot into Apple firmware or Limine: skewed logo or panel stuck in dual-tile mode? Does the lean shutdown handoff help or hurt on iMac20? | Warm reboots, camera on the boot logo | D3 |
| U13 | Does the panel support DP sink CRC (DPCD `0x246` TEST_CRC) on both tiles, for the rig's per-tile check? | DPCD read | R1 |
| U14 | Does Hyprland accept the single 966.5 MHz 5120×2880 mode and the synthetic EDID cleanly at scale 2? | D2 | D2 |

---

## 7. Risks, ranked

1. **Genlock on DCN is new ground** (U5, A4/A5). DCN's per-frame reset has never been
   enabled by default upstream (the AMD VSDB "multi_sync" TODO). Mitigation: default to
   one-shot + measured re-align, which uses only code paths DCN already runs for
   multi-monitor sync.
2. **Tile metadata on iMac20** (U1). If the EDIDs lack tile blocks, the stitch silently
   falls back (with retina 14) or shows zero modes (without it). Mitigation: retina 14 is
   in the base; A1 can supply the geometry from DMI.
3. **Right half black or stale after a modeset** without retina 15. It is in the base.
4. **Kernel churn:** the stack targets 7.1/7.2. Retina notes that 7.3 moves code into
   `amdgpu_dm_connector.c`. That cost recurs on every kernel bump until upstream (phase U).
5. **Bandwidth:** low risk. The scary log line is DML search noise (§4.4). The real check
   is memory-clock idle (U9).
6. **Shared-DC edits** (A5, A6) could affect non-Apple DCN users if the gate is wrong.
   Mitigation: every DC-level change keyed on a `dc->config` flag that only the DMI quirk
   sets.

---

## Appendix: reproduction of the R0 checks

```sh
# scratch copy of the module sources (original tree untouched)
mkdir -p ~/src/5k-ref/scratch-r0/base
git -C ~/Sysadmin/patches/amdgpu-imac20-2-vpost/build/linux-7.2.9 archive HEAD \
    drivers/gpu/drm/amd include/drm | tar -x -C ~/src/5k-ref/scratch-r0/base
# then, per stack: cp -a base v-NAME; patch -p1 -d v-NAME --forward < each.patch
```

The 7003 boot is boot −1 in `journalctl --list-boots` as of 2026-10-10 (20:00 boot; it
will rotate out). Lines quoted above: `imac5K_support attached`, `set_pixel_clock_v7 …
4832500, tg = 0/1`, `Link(1) training attempt 1 of 4 failed … fail reason:(8)`.
