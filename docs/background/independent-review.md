# Review: 5K-stitch PoC series (b8d020e..02906a9), 2026-10-10

Adversarial review of the untested 23-patch series on branch `5k-stitch-poc`. I read
DESIGN.md, README.md and the whole diff `b8d020e..HEAD`, with most attention on the
hunks that DCN 2.0 reaches through code paths that DCE never used. I cross-checked the
stock 7.2 DC code that the series calls into: `program_timing_sync`,
`dcn10_enable_timing_synchronization`, `optc1_*_reset_trigger`,
`create_stream_for_sink`, the DCN cursor path, the IP-block order and PSR gating. I also
used the archived 7003 boot log and the live eDP EDID (read-only from sysfs). Nothing
was built, loaded or modified. sparse and smatch are not installed. checkpatch on 01–17
reports style warnings only (block comments, long lines).

**Bottom line.** For a PoC the DCN adapter (18–23) is small and mostly right. The
biggest risks are in the reused stack: code that ran on DCE but takes a different path
on Navi 10. Then come four hardware unknowns that all have to come out right before
D1 can show anything. Three of those four can be answered *before* this series ever
boots, at no risk (see "Do this first").

Severity: **blocker** means it will probably stop D1 or make its result meaningless.
**Likely-bug** is wrong code that will probably show up on hardware. **Risk** depends
on an unknown, or bites later (D2/D3). **Nit** is minor.

---

## Do this first (zero-risk, before any dev boot)

Most of the D1 outcome depends on facts that one boot of the stock `linux-t2` module can
show. That module already carries 7003, and it gave the two-output result on
2026-10-10. Nobody captured that boot's EDIDs. On the next such boot, record:

```
for c in eDP-1 DP-1; do cat /sys/class/drm/card1-$c/edid > ~/Sysadmin/…/$c.edid; edid-decode < …; done
```

Check for: a DisplayID **tiled-topology block** on both EDIDs (2×1, 2560×2880,
`tile_h_loc` 0 and 1, one shared group ID). This is U1. Check the **slave's panel ID**,
which must be in the `AE01…AE32` list (U2, and it also decides A3). Check whether the
root's flipped ID is AE32. Also read DPCD `0x060` (DSC support) on both links through
`/dev/drm_dp_aux*`, read-only (see L4). If the tile blocks or the slave ID are wrong,
D1 cannot work as built, and the fix is new code (B1), not a test campaign.

---

## Blockers

### B1. Four hardware unknowns gate the stitch, and none of them is proven by D0b's checks as written
*commits 02 `8404f2a86`, 14 `a3dc7e572`, 16 `18a81b258`; `amdgpu_dm.c:amdgpu_dm_tiled_stitch_active()`, `amdgpu_dm_connector_tiled_stitch_geometry()`, `amdgpu_dm_reprobe_tiled_root_after_slave()`, `amdgpu_dm_is_tiled_slave()`*

The stitch shows the 5120 mode only if all of these hold:
1. the root connector has `num_h_tile == 2`, `num_v_tile == 1` and tile sizes, which
   only a DisplayID tile block in the **re-read** root EDID provides;
2. the slave connector has its own tile block with `tile_h_loc != 0`. Without it,
   `amdgpu_dm_reprobe_tiled_root_after_slave()` returns before it re-reads the root, so
   the root keeps the **AE31 4K EDID** that was read *before* the `0x4F1` wake;
3. the slave's own EDID panel ID is in the `apply_edid_quirks()` list, because
   `tiled_stitch_active()` requires `slave->local_sink->…apple_tiled_slave`;
4. both tiles report the same DRM tile group.

Only the root ID (AE31, `Model: 44593` in the live EDID) is known today. README D0b check 2
("wired tile pair: root link[0] <-> slave link[1]") does **not** test (3): the root wires
by connector signal and link index, whatever the slave's ID is.

Failure scenario (D1, any of 1–3 false). There is no stitched mode. The slave connector
is still marked `non_desktop`, because that keys on the role, not on the stitch being
active (see L2). The panel is latched into dual-tile mode while the root either drives
the 4K identity timing (garbled or black left half) or, if the root was re-read, a lone
2560×2880 left half. Either way the result looks like a stitch bug but is a metadata
problem.

Fix: run "Do this first". If iMac20 lacks tile blocks, A1 must supply the geometry from
the DMI match, and the root's EDID read must happen *after* the latch, as 7003 does.
That is roughly 60–120 lines, and it is the one piece of the reference design that does
not carry over. Add a D0b log check that names the slave's product ID and `has_tile`
for both connectors.

### B2. The peer cursor is programmed on a DCN-only path, without `dc_lock`, mid-commit. The DCE testing never exercised it
*commit 02 `8404f2a86`; `amdgpu_dm.c:amdgpu_dm_update_cursor()` → `amdgpu_dm_tiled_program_peer_cursor()`*

In 7.2, `amdgpu_dm_commit_planes()` calls `amdgpu_dm_update_cursor()` whenever
`amdgpu_ip_version(DCE_HWIP) != 0`. That is true for every DCN part, Navi 10 included.
The stock function only *stages* the root cursor (`dc_stream_set_cursor_*` +
`update->cursor_*`), and DC programs it later under `dc_lock`. The stitch hook instead
calls `dc_stream_program_cursor_attributes/position()` on the peer **immediately**.
That writes hardware with no `dc_lock` held, from the plane loop, before the root's own
update. `DCE_HWIP` is 0 on Tonga, Polaris and Vega10 (`amdgpu_discovery.c` sets it only
for DCN parts). So on every machine the reference stack was tested on, the cursor went
through `amdgpu_dm_plane_handle_cursor_update()`, which does hold `dc_lock`. **This
path has never run.**

Failure scenarios: (a) `dc_stream_program_cursor_position()` walks
`dc->current_state->res_ctx`. A concurrent full update on another CRTC (Thunderbolt
monitor), or link-loss recovery, can swap or free that state, which gives a
use-after-free window. (b) The cursor shows a frame early or late on the right half,
or flickers at the seam, because the peer write is not double-buffered with the root
update. (c) On the commit that creates the peer stream, the cursor is programmed before
the peer pipe has a plane.

Fix: take `dc_lock` around the peer calls in `amdgpu_dm_update_cursor()`. Better, stage
the peer cursor (`dc_stream_set_cursor_*` on `stream_peer`) and pass it in
`tiled_peer_stream_update`. Cursor-only commits need the same, since they have no peer
surface update today. D1 excludes the cursor, but Hyprland draws a hardware cursor from
the first frame, so this runs on the first D1 boot whatever the plan says.

---

## Likely bugs

### L1. With `tiled_genlock=1`, every modeset commit disarms the per-frame reset it just armed
*commit 22 `66c02e4a9` (A5), interacting with stock `dc.c:dc_commit_state_no_check()` / `program_timing_sync()`*

The order in DC is `enable_timing_multisync()`, then `program_timing_sync()`. In a
modeset the slave pipe is still blanked, so `program_timing_sync()` keeps it in the
group and runs `dcn10_enable_timing_synchronization()`. That calls
`optc1_enable_reset_trigger()`, which overwrites `OTG_TRIGA_CNTL`, waits, and then
calls `optc1_disable_reset_trigger()`. That writes `OTG_TRIGA_CNTL = 0` and clears
`FORCE_COUNT_NOW`, which tears down the per-frame trigger A5 armed microseconds
earlier. Per-frame reset is then armed only later, by the 250 ms resync worker's
`dc_trigger_sync()` on the lit pair (where unblanked pipes drop out of the one-shot
group). The A8 log line says "genlock=per-frame" even when the reset is not armed.

Effect: the U5 comparison (one-shot against per-frame) measures a hybrid of the two.
Also, when the resync worker runs, `wait_for_reset_trigger_to_occur()` can spin for up
to 10 frames (~170 ms) under `dc_lock`. Fix: for the Apple pair under
`apple_tiled_genlock`, skip the tile pair in `program_timing_sync()`, or re-arm the
per-frame reset after it. Log `OTG_TRIGA_CNTL` read-back in A8 so the log shows what is
armed, not what was asked for.

### L2. The slave is hidden as `non_desktop` even when the stitch is inactive
*commit 02 `8404f2a86`; `amdgpu_dm_connector_get_modes()`, `amdgpu_dm_tiled_stitch_edid_connector_update()`*

Both places key on `amdgpu_dm_link_is_tiled_stitch_slave()`, which is role-wired and
`tiled_stitch != 0`. They do not check `amdgpu_dm_tiled_stitch_active(root)`. Whenever
the stitch gate fails (B1, a transient slave loss before the latch), Hyprland sees only
the left tile and nothing can drive the right one. Fix: hide the slave only while
`tiled_stitch_active(root)`. Otherwise fall back to the D0b behaviour of two connectors.

### L3. A3 keys on the slave's own product ID, which is unknown
*commit 20 `23ed0f232`; `dc/dc.h:dc_link_is_apple_tiled_dp_panel_mode()`*

Once the slave has a sink, the check uses `link->local_sink` (the slave's EDID), not the
root's. If the slave reports anything other than AE31/AE32, the tile goes back to **eDP
panel mode**, the opposite of what 7003 did and what A3 meant. The likely symptom is
"trained but dark" (`0x205 = 00`), and the README's advice ("drop these two lines")
would then be the wrong move. Fix: on `dc->config.apple_imac20_tiles`, return
`true` for the slave regardless of ID, and log the slave's product ID in A8.

### L4. DSC policy runs on the 966 MHz full-width timing before it is halved
*commit 02 `8404f2a86`; `amdgpu_dm.c:create_stream_for_sink()`*

`update_dsc_caps()` and `apply_dsc_policy_for_stream()` run at lines ~8841–8843, and
`amdgpu_dm_tiled_stitch_apply_tile_timing()` runs afterwards at ~8855. DCE has no DSC,
so this ordering has never been exercised. If the iMac20 eDP sink advertises DSC (DPCD
`0x060` bit 0, unknown), DC computes a DSC config for a 5120-wide, 966 MHz stream over
HBR2×4, sets `timing.flags.DSC`, and the halving then leaves a DSC config for the wrong
width. That config is cloned onto the peer too. Then either `mode_valid` prunes the only
mode (zero modes, black panel) or the stream is garbage. Fix: apply the tile timing
before the DSC step, or skip DSC on a stitched root. Check `0x060` first. If DSC is
unsupported this is latent and harmless.

### L5. The link-health escalation relies on the compositor honouring `link-status`, and Hyprland very likely doesn't
*commit 16 `18a81b258` (and 17); `amdgpu_dm_tiled_request_full_modeset()`*

After two failed second-tile recoveries, or a bad root, the kernel sets the root's
`link-status` to BAD and sends a hotplug event, expecting a full modeset from userspace.
KWin and Mutter do that. I know of no such handling in wlroots or aquamarine (verify).
On Hyprland nothing resets the property and nothing modesets, so the right half stays
dark until a DPMS cycle or a reboot. The stitched mode list is also latched (L6), so
there is no smaller mode to fall back to. Note: the 7003 boot already showed one link-1
`LINK_LOSS` at 17.7 s. Fix: do the recovery in the kernel, as
`amdgpu_dm_tiled_slave_sink_gained()` already does: run `dm_force_atomic_commit()` on
the root with `mode_changed`, from a worker, bounded.

### L6. A permanently lost second tile leaves a 5120 desktop with half of it invisible
*commit 14 `a3dc7e572`; `amdgpu_dm_tiled_stitch_active()` (`tiled_stitch_latched`)*

Once latched, the stitch stays active while the root's cached EDID keeps its tile
fields. Those fields are restored by `restore_tile_state_hidden()` even on a NULL or
4K re-read. If link 1 stops training for good, `create_tiled_peer_stream()` returns
NULL ("right tile will be dark"), the root keeps the only (5120) mode, and the root
plane shows only its left 2560 columns. Nothing ever offers 2560×2880 or 4K again during
that boot. This is deliberate upstream: it keeps the compositor's output identity
stable. But it means the "falls back to a usable mode" requirement is **not met**. Fix:
after L5's recovery fails, unlatch, re-probe the root and send a hotplug event. Hyprland
then gets a smaller mode with nothing hidden.

### L7. DPMS-on and GPU-reset bring-up probably train the peer before the root powers the panel
*commits 02 + 16; `create_tiled_peer_stream()` added before the root in `dm_update_crtc_state()`; `link_dpms.c:enable_link_dp()` root wake*

The peer stream is added to the context first, so on a cold pipe pool it gets pipe and
OTG 0. `dce110_apply_ctx_to_hw()`, which DCN 2.0 also uses, enables pipes in ascending
order, so link 1 trains *before* link 0's `enable_link_dp()` powers the eDP panel and
re-arms `0x4F1`. retina 16's comment ("whatever the pipe order") is only true if the
root enables first. At first boot the panel VDD is probably still on from detection. On
DPMS off, which Omarchy's hypridle triggers, the latch clears with the root's D3 (as
measured on iMac15,1). On the following DPMS on, the peer's AUX is dead:
`link_apple_5k_slave_aux_ready()` tries to wake the panel through a powered-off root,
fails, and takes 4 training attempts (aborted), each up to 3×20 ms plus AUX timeouts,
under `dc_lock`. Recovery is then left to the post-commit link-health pass (+250 ms) and
L5. Expected symptom: slow wake, with the right half black or flashing for about 1 s,
and sometimes stuck (L5/L6). Fix: add the root stream first, or power the root and
pulse the latch from the peer's enable path when the root's VDD is off.
**Check in D1:** which OTG each tile gets (A8 logs it).

---

## Risks

| # | Where (commit) | Scenario | Suggested fix |
|---|---|---|---|
| R1 | `dm_helpers_tiled_pair_allowed()` (19 `32194f0`) | Pins the root to `link_index 0`. The evidence is decent: in the 7003 log `link(0)` trains first with `spread = 10` (SSC, typical for eDP) and `link(1)` is the tile. But enumeration order is a VBIOS artefact. If the root isn't link 0, the pair silently never wires. That fails safe, but D0b would look like "no 5K" | Pin only the slave (`link_index 1`), or better pin by HPD source or connector object ID, which are hardware-fixed. Log the reject |
| R2 | `amdgpu_dm_tiled_measured_realign()` (13, used by 21) on a lit pair | `dcn10_enable_timing_synchronization()` resets the slave OTG at the master's vsync, which truncates one frame on the slave's DP stream. The TCON may blank that tile for a frame, or worse, lose dual-tile lock. The DPG `2*h+1` reprogramming is harmless on an unblanked pipe (DPG is inactive). Which half jumps depends on pipe order: master = lower pipe, probably the **peer**, so it is the *root* (left) that jumps, not the right as README says. Holds `dc_lock` for up to 10 frames | Expected and gated by tolerance. Make D1 count `re-aligned` lines and check camera frames around them |
| R3 | `amdgpu_dm_tiled_phase_delta()` (13) | `(2b − a − a2)/2` averages `a` and `a2` without wrapping. If the frame wraps between those two reads (~1 in 3–5 k reads), the result is ±v_total/2, which falsely triggers a re-align (R2's glitch). The worker only runs after commits with global state (not on plain flips: `dc_state` is NULL there), so this is rare, perhaps once per 10³ modesets | Unwrap `a2` relative to `a` modulo `v_total` before averaging. Same fix makes `amdgpu_dm_tiled_log_phase()`'s sign trustworthy. The sign convention itself is right: root first, positive = peer ahead |
| R4 | Peer stream link bandwidth (02) | The peer is cloned from the root and never goes through `dc_validate_stream()`/link-bandwidth validation (only global validation). If link 1 trains at a fallback rate, the stream oversubscribes the link: black or garbled right tile. retina's `link_ready` checks lanes == 4 but not the rate | Validate the peer against `link->verified_link_cap` in check, and require HBR2 in `tiled_link_ready()` |
| R5 | Seam updates (02) | Root and peer are separate `dc_update_planes_and_stream()` calls. If the peer misses the vblank, userspace has already been sent `flip_done` from the root's interrupt, so the compositor reuses a buffer the peer is still scanning. That is a one-frame corruption on the right half, not just tearing | D2: GSL flip-lock (DESIGN §4.5), or wait for the peer's flip in the root's pflip handler |
| R6 | Resume (06, 07) | S3 resume re-detects the root after the latch cleared, so `dc_edid` reverts to the AE31 4K identity. Tile fields are restored from the saved state, but the root `dc_sink`'s caps now describe a 4K panel driving 2560×2880. Probably benign on DC. This machine's suspend is already unreliable (`sleep-failure.md`) | D3, test explicitly |
| R7 | GPU reset (07) | The mode-1 reset drops both PHYs. Restore goes through `dm_gpureset_commit_state()` with the peer first (L7), then link health and L5 | D3. retina 17 (HPD during reset) was skipped; revisit |
| R8 | A6 missing (22) | Documented: a stale `OTG_TRIGA` on an OTG later reused for a Thunderbolt monitor. The L1 interplay makes it rarer on modeset, but the worker re-arms it | Keep `tiled_genlock=1` off default boots, as planned |
| R9 | No VRR guard on the stitched root (02) | The synthetic EDID drops the range descriptor, so `vrr_capable` should be 0. But a forced VRR (or freesync-video mode) would adjust v_total on the root only and immediately shear the seam | Reject `vrr_active` on a CRTC with `stream_peer` in `amdgpu_dm_crtc_helper_atomic_check()` (5 lines, next to the async-flip reject) |
| R10 | Plymouth | The root's TILE blob is hidden but the slave's is not (`non_desktop` only). Plymouth has its own tiled-head logic and may try to light DP-1 by itself. That gets `-EINVAL` from `slave_reserved`, which is harmless but noisy. fbdev is fine: `drm_client` skips `non_desktop` | Watch the D1 Plymouth log |
| R11 | Ungated behaviour changes on non-iMac20 hardware (09–11, 13) | The DCE paths are not panel-gated: the DCE 10/11 GSL register-address fix (`dce110_timing_generator.c`, which changes multi-display timing sync on **every** Tonga/Polaris/Carrizo user), DCE 12 master-first, `dce120 .enable_crtc_reset`, and DCE < 12 `force_timing_sync` rerouting. They are inert on Navi 10, so there is no runtime risk to *this* machine, but they matter for upstreaming | Split them into standalone DCE fixes if they are ever sent upstream |
| R12 | Logging (01) | `APPLE5K: link-health build=…` prints at every amdgpu init on every machine | Gate on `apple_imac20_tiles` or the role flags |

On the README's specific questions:
- **`amdgpu_dm_per_frame_crtc_reset_supported()` disabling debugfs `force_timing_sync`**
  on iMac20 (21): confirmed, harmless. It only affects the debugfs knob, which nobody
  uses here.
- **`OTG_TRIGA_SOURCE_PIPE_SELECT` takes an OTG instance** (22): confirmed. Stock
  `optc1_enable_reset_trigger()` writes `grouped_pipes[0]->stream_res.tg->inst` into
  the same field.
- **DMI table** (18): `DMI_EXACT_MATCH`, entries inserted before the existing `{}`
  terminator. Correct.
- **A7 = 0 lines** (PSR): confirmed. `amdgpu_dm_set_psr_caps()` returns early without
  `dc->caps.dmub_caps.psr`, and Navi 10 also has no DMCU firmware loaded.
- **A1 gate**: effective. No path reaches the role flags on a non-DMI DCN machine, and
  `tiled_stitch=0` disables every stitch path. The lean core (wake, link health on
  resume) still runs with stitch=0, which is intended for D0b. One README claim is
  slightly off: A3 does not check `apple_imac20_tiles` itself. It is only reachable
  through the role flags, so it is gated in effect.

## Things checked and found OK

- Refcounting of `stream_peer` and `dc_state_peer`. They are duplicated and destroyed
  in `amdgpu_dm_crtc.c`/`amdgpu_dm_plane.c`. Create: one ref for the CRTC state, one
  for the context, and the create ref is dropped. A failed `dc_state_add_stream` drops
  both. A late `goto fail` leaves the peer in a context that is thrown away. The
  disable path mirrors the stock root release. The cached-state release on resume
  (06). I found no leak or double release.
- Worker lifetime. `amdgpu_dm_fini()` sets `stopping` under `dc_lock` and then calls
  `cancel_delayed_work_sync()` before `dc_destroy`. Shutdown does the same. Reset makes
  the worker re-queue itself. The worker never re-arms itself after `stopping`.
- **Probe-defer coexistence.** IP order on Navi 10 is … SMU (3439), then display (3444).
  `amdgpu_dm_init()` runs from `dm_hw_init()`, so a first-probe SMU `-ETIME` unwinds
  before any DM state exists: no latch, no worker, no `INIT_DELAYED_WORK`. The extra
  `drm_client_dev_hotplug()` runs only after a successful load. Static state that
  survives the deferral: `quirk_entries` (set-only, recomputed every probe) and a few
  `*_once` log flags. None of it affects behaviour. The `gpu-boot-check` rebind runs
  only after a failed probe, so it is in the same position.
- Gating of the DCN shared-code edits in A5 (`apple_tiled_genlock`, set only on DMI +
  `tiled_genlock=1`).
- Async flips are rejected on a stitched CRTC. Overlay and scaled planes are rejected
  in check (`TEST_ONLY` included), so Hyprland falls back to compositing. Colour:
  CRTC colour changes force a plane reset, and the peer plane is rebuilt with full
  colour state. Stream regamma, CTM and CSC are copied, which covers hyprsunset (CTM).
  DCN 2.0 has no 3D LUT to miss.
- Pipe split, ODM and bandwidth: two 483 MHz streams under `MPC_SPLIT_AVOID_MULT_DISP`.
  The 7003 boot already passed DML with this pair.

## Nits

- **Authorship (all 23 commits).** Git author is the owner. The real authors (Pronkin as
  `mr_projects`, erik2 for the stitch layer, br1uhnz for retina, Ahmad as the original
  copyright holder) appear only in `[upstream: …]` lines. Don't change it in this PoC
  branch, but before anything leaves the machine, fix it per commit, for example
  `git rebase --exec 'git commit --amend --no-edit --author="Mark Pronkin <…>"'` scoped
  to the right commits, or `git filter-repo --commit-callback`. That makes
  `format-patch` emit the correct `From:`, and `git send-email` then adds the
  in-body `From:` line. Add `Co-developed-by:`/`Signed-off-by:` pairs. Note the reused
  patches carry only `Not-Yet-Signed-off-by: mr_projects`, so **there is no DCO**. They
  cannot go upstream until the authors sign off. Also make the AI trailer consistent:
  the probe-defer patch uses kernel-style `Assisted-by:`, this series uses
  `Co-Authored-By:`, and the reused retina commits carry a second, foreign
  `Co-Authored-By: Claude Opus 5 (1M context)`.
- `dm_update_crtc_state()` comment (02) refers to a "keep-stream gate ahead of this
  branch" for CRTC-off that does not exist. CRTC-off removes both streams. Fix the
  stale comment.
- The "logical modeset" shortcut (02, guarded by 05) is effectively dead now: the
  disable pass has always removed the stream by the time the enable pass checks it.
  Consider deleting it rather than carrying it.
- DRM vblank timestamps use the doubled mode's pixel duration with a tile-domain
  `hpos`. The error is under one line. Cosmetic.
- checkpatch style warnings in 01–17 (block comments, lines > 100 columns).

---

## Upstreamability and maintenance cost

- **The stitch layer (02, 14, 15, and most of 16) will not go upstream.** It makes up
  ~2.3 k of the 3.2 k lines: an in-kernel synthetic EDID generator, a hidden TILE
  property, `non_desktop` applied to half of a real panel, and the kernel splitting a
  framebuffer across CRTC-less streams. DRM's position is that tiled displays are
  userspace's job. Pronkin's own patch header says "upstream will not take this layer,
  the lean core is the mainline candidate".
- The lean core plus A1–A3 is a plausible upstream candidate (~700 lines), but it needs
  the authors' DCO and an AMD reviewer who is willing to take panel-ID-plus-vendor-DPCD
  quirks.
- **Kernel churn.** `amdgpu_dm.c` gets +2 313 lines spread over ~40 hunks in
  connector, EDID, mode, atomic-check and commit paths, the hottest file in the
  driver. 7.3's move of connector code to `amdgpu_dm_connector.c` will make roughly a
  third of those hunks fail to apply and need manual porting, about 1–2 days once.
  After that, expect 0.5–1 day per minor kernel bump, plus re-running the D1/D3
  campaigns, because nothing guards these paths against upstream refactors. `linux-t2`
  carries 7003, which conflicts with this model, so building from its source means
  reverting 7003 first (DESIGN §3).
- The long-term alternative is tiled-output support in aquamarine/Hyprland on top of
  the lean core alone. It is bigger up front, but needs no out-of-tree kernel code
  beyond ~700 upstreamable lines.

---

## Probable complexity (independent estimate)

**Is the PoC approach sound?** Yes, *as a feasibility probe*. Reusing the stack that
runs on DCE is the cheapest way to learn whether the iMac20 TCON accepts the lean
core's bring-up and whether DC on DCN drives two tile streams from one CRTC. The DCN
delta in 18–23 is small and mostly correct. Two parts of the reasoning I would not
accept. "The stitch is engine-agnostic, so it is tested" is wrong: B2, L4 and L7 are
paths that DCE never exercised. And the README's "zero porting" understates the work
still to come, which is debugging, not compiling.

**Remaining work to D1** (one connector, 5120 mode, correct `modetest` pattern on
camera, 20 clean boots). This does not count the R1 rig, which D1 needs and which does
not exist yet.
- *Favourable branch* ("Do this first" shows tile blocks, slave ID in the list, no
  DSC): fix B2, L1, L3 and L4 before the first D1 boot (~50–100 lines), then expect
  **5–15 dev boots** to reach a correct pattern. Most of the time goes to seam alignment
  (R2/R3) and link 1's first-bring-up flakiness (`LINK_LOSS` was already seen in
  7003). Then the 20-boot gate. Roughly **1–2 weeks** of part-time agent work once the
  rig runs.
- *Unfavourable branch* (no tile block on iMac20): DMI-supplied geometry, a reordered
  root EDID read, and reworked latch/fallback logic. That is new design inside the
  reference stack's least-tested area. **+150–300 lines, 2–4 weeks**, and the reference
  stack's fixes then no longer apply cleanly.

**Remaining work to D3** (suspend, GPU reset, warm reboot, probe-defer, 24 h soak, mclk
idle, 30-boot campaign): this is where most of the cost is.
- Must-do code: kernel-side recovery instead of link-status BAD (L5), unlatch and
  fallback (L6), root-first bring-up order (L7), A6 trigger teardown, R4 validation,
  and probably GSL flip-lock (R5) for D2. My estimate is **300–800 more lines**, with
  several hundred boots across campaigns.
- Suspend is already unreliable on this machine, so D3's "suspend 20× clean" gate
  depends on `sleep-failure.md` and not on this series.
- **4–8 weeks** part-time, with a real (~30–40 %) chance of a hard stop at D2/D3 on
  something only the TCON knows: DPMS-wake latch behaviour, warm-reboot handoff, or
  intermittent second-tile link loss. The DCE projects took months of iteration on
  exactly those, with a human at the screen.
- After D3, the permanent cost is the per-kernel rebase plus campaigns (see
  maintenance), for a single machine.

My verdict: worth running **D0b** (cheap, low risk, answers B1). Make D1 conditional on
"Do this first" coming back favourable. Make a deliberate decision before committing to
D3, because that is the expensive half, and its cost recurs.

---

## First-boot prediction

**D0b** (`amdgpu.tiled_stitch=0`), most likely first:
1. (~45 %) Both tiles detect and train. The root is re-read to the tile identity, and
   Hyprland shows two 2560×2880 outputs, as 7003 did. Possibly a shear on one, from
   independent per-output modesets, until the second output's one-shot aligns it. The
   A8 lines appear. This answers U1 and U2.
2. (~25 %) The slave has no tile block, so the root is never re-read. The panel is
   latched dual-tile while eDP-1 drives the AE31 4K timing: a garbled or black left
   half, and DP-1 at its own EDID mode. The machine is reachable over SSH.
3. (~15 %) The right tile trains but stays dark (`0x205 = 00`), from the panel mode
   (L3) or AUX timing on link 1.
4. (~10 %) No DMI match, or wiring rejected (R1): plain 4K, as today. Cheap to see from
   the missing `APPLE5K:` lines.
5. (< 5 %) Oops or hang. The rig's panic cmdline turns it into a reboot.

**D1** (defaults), most likely first:
1. (~35 %) The stitch does not engage (B1). One left half, DP-1 hidden (L2), or a
   garbled 4K. The logs show `never exposed its tile block` or no `exposed only
   stitched mode` line.
2. (~25 %) The stitch engages: 5120×2880, both halves lit at Plymouth and in Hyprland,
   link-health PASS. But the cursor misbehaves on the right half (B2), and there is
   occasionally one `re-aligned` jump. This is the "it basically works" outcome.
3. (~20 %) The stitch engages but the right half is black: peer training, the L3 panel
   mode, or link loss. The logs show link-health FAIL and link status BAD, which
   Hyprland ignores (L5). The state persists for the boot (L6).
4. (~10 %) Zero modes or a black panel, from L4 (if the sink has DSC) or a
   geometry/validation mismatch.
5. (~10 %) Stalls of seconds under `dc_lock` (AUX retries and 10-frame waits), with
   Plymouth or the LUKS prompt visibly slow, or an oops in the unlocked cursor path
   (B2). Only a minority of these should be outright hangs.

Whichever happens, the first D1 boot answers which OTG each tile gets (L7), the slave's
product ID (L3), and whether `0x205` reads `01`. Put all three in the rig's
must-capture list.

---

## Addendum 2026-10-11: "Do this first" done (run 017), favourable branch

Dev boot of the stock `linux-t2` 7.2 amdgpu (carries 7003; sha256 `a924f84f…`) through
the rig, first probe OK (no rebind). Evidence: `runs/017/collect2/` (EDIDs, `edid-decode`,
DPCD dump, `modetest`).

| Question | Answer |
|---|---|
| U1: DisplayID tile block on both EDIDs? | **Yes.** Both: 2×1 tiles, 2560×2880, single enclosure. eDP-1 (root) at location (0,0), DP-1 (slave) at (1,0). One group: vendor `APP`, product 44595 (`0xAE33`), serial 2212588592 |
| U2 / A3 / L3: slave's panel ID in `AE01…AE32`? | **Yes: `AE32`** (Model 44594). The root, re-read after the `0x4F1` wake, is also `AE32` (its 4K identity on a known-good boot is `AE31`, 44593). So L3's "slave reports something else" does not happen here |
| L4: DSC (DPCD `0x060` bit 0)? | **No**, `0x060` = `00` on both links. L4 is latent and harmless on this panel |
| `0x205` on link 1 (A3) | `01` on both links. Lanes fully trained (`0x202–0x203` = `77 77`), interlane aligned |
| Link caps (`0x000`) | Both: DPCD 1.2, HBR2, 4 lanes, enhanced framing, TPS3 |
| What the panel shows (owner) | Two correct desktops side by side; the cursor crosses the seam smoothly. Hyprland: two 2560×2880 outputs at scale 1.6 |

Still open:
- `modetest -c` prints an empty blob for the DRM `TILE` property on both connectors.
  The PoC keys on the connector's parsed tile fields (`has_tile`, `num_h_tile`, …), not
  the property, so this probably doesn't matter, but D0b should log `has_tile` per
  connector (B1's suggested check) to confirm.
- B2 (unlocked peer cursor) still has to be fixed before D1. L1 matters only with
  `tiled_genlock=1`.

Consequence for the plan: D1 is worth trying. D0a (probe-defer + 7003 rebuilt) is
mostly answered by run 017 (two correct outputs, rig sees both halves); D0b is next.

## Addendum 2026-10-12: L5 and L6 fixed (candidate build, runs 090–101)

- **L5:** confirmed on hardware that Hyprland ignores `link-status` BAD. `0033` replaces the
  escalation with an in-kernel `dm_force_atomic_commit()` on the root from a worker (at most 2,
  2 s apart, inside the existing 3/min and 12/boot budget, skipped while the CRTC is off or
  during suspend/reset). With `tiled_fault=5`: FAIL → kernel modeset 1/2 → PASS.
- **L6:** the review's suggestion (unlatch, offer smaller modes, hotplug) was built first and
  failed on hardware: Hyprland kept the 5120 mode, and the next DPMS-on was rejected, which
  left the panel black. `0034` instead leaves everything userspace sees unchanged and drives the
  root tile alone, with the whole mode squeezed 2:1 into it. The TCON, with the `0x4F1` latch
  still set, stretches the lone root tile back over the panel. The owner saw the complete
  desktop with correct proportions, through a DPMS cycle and an S3 resume. Re-latch never
  triggers (no slave HPD while link 1 is undriven), so the fallback lasts until reboot.
- **Wake glitch (L7 residue):** not a link failure. See `0036` and runs/README 090–091.

