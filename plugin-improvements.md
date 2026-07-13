# Plugin Improvements — future work INSIDE our plugins

Running list of enhancements and fixes we would code **inside the plugins
themselves** — as opposed to platform/infra work outside them (pywaggle, SES,
WES, ECR, Beehive, node/network), which lives in `Infra-problems-to-fix.md`.

The dividing line: if the fix is a change to plugin source in one of our repos,
it belongs HERE. If it's a change to the shared library or the platform, it
belongs in `Infra-problems-to-fix.md`. Some needs are cross-cutting (a platform
primitive PLUS a per-plugin change); those appear in both, each scoped to its
side, with a cross-ref.

Plugins in scope: **image-sampler2** (github.com/flint-pete/image-sampler2),
**bioclip** (sage-bioclip), **yolo** (sage-yolo), **birdnet**.

Companion docs: `Infra-problems-to-fix.md` (platform/infra), `pywaggle2-design.md`
(the pywaggle redesign these plugins would consume),
`image-sampler2/readiness-gap.txt` (image-sampler2's usability blockers),
`image-sampler2/docs/imagesampler.flint.analysis.txt` (image-sampler2 locked
design; Part 4 OPEN + Part 5 DEFERRED are the source of its items below).

Legend: [FEATURE] new capability · [FIX] correctness · [DECISION] needs Pete's
call. Priority: P1 blocks usability / P2 limits reach / P3 polish.

Last updated: 2026-07-07.

================================================================================
image-sampler2
================================================================================

## IS-1. [FEATURE][P2] Multi-vendor native-still: add Hanwha SUNAPI + Mobotix
Only Reolink (`cmd=Snap`) is implemented in `acquire.py::build_reolink_snap_url`.
The metadata-preserving still endpoints for the other fleet vendors are spec'd
(design 4.3, OPEN) but not built:
- **Hanwha/Wisenet SUNAPI**: `http://IP/stw-cgi/video.cgi?msubmenu=snapshot&
  action=view&Profile=P&Channel=N` — returns a JPEG only if an MJPEG profile is
  configured, else errors. Mirrors the Reolink raw-bytes + EXIF-inject path
  exactly; a `build_hanwha_snapshot_url()` beside the Reolink one.
- **Mobotix**: `/record/current.jpg`, `/cgi-bin/image.jpg?...`,
  `/control/event.jpg?sequence=head` — authors the rich M1IMG fingerprint.
Blocked on real hardware to verify (which Mobotix URL; whether WSN Hanwha have an
MJPEG profile; what EXIF Hanwha snapshots actually carry). This is image-
sampler2's slice of the pywaggle2 acquisition ladder — see pywaggle2-design.md §1.

## IS-2. [FEATURE][P2] OpenCV/RTSP fallback (the acquisition floor)
`acquire.py::capture_opencv_fallback` is a stub that raises. It is the lowest
rung of the acquisition ladder: for stream-only cameras (or H.264/H.265 RTSP
where no per-frame metadata exists anyway), decode one frame -> JPEG, label
`acquisition_path="opencv-reencoded"`. Without it, image-sampler2 cannot talk to
any camera lacking a native still endpoint. NOTE (Pete's steer 2026-07-07):
lower priority — WSN/recent cameras are all IP/RTSP with snapshot endpoints; the
USB/stream-only case is a future enhancement. The real focus is metadata-
preserving RTSP snapshots (IS-1), not this floor.

## IS-3. [DECISION][P3] Resize/quality vs never-re-encode (design 4.5, OPEN)
Decide how `--jpeg-quality`/`--resize` coexist with the raw-native mandate.
Options: (a) apply only on the re-encode/fallback path; (b) produce a SEPARATE
derived resized image alongside the untouched raw; (c) drop entirely (raw only,
consumers resize downstream). Currently raw-only — a defensible permanent answer.
Needs Pete's call; never silently re-encode a native-raw image.

## IS-4. [FEATURE][P3] --from-cache time-window selectors (design 5.2, DEFERRED)
v1 `--from-cache` selects NEWEST only. Add `--closest-before-timestamp <ns>`
(newest with capture_ts <= T) and `--closest-after-timestamp <ns>` (oldest with
capture_ts >= T) for cross-node event correlation. Selection uses the capture-ts
prefix in the v2 filename. Corner cases: no image satisfies the bound
(fail-fast); ties; eviction between select and upload (ENOENT -> "gone").

## IS-5. [FEATURE][P3] Cache discovery / announcement (design 5.1, DEFERRED)
Instead of consumers relying purely on convention, have the producer ANNOUNCE its
cache: publish a Waggle record (cache_path, camera, config summary) and/or write
a `manifest.json` at a well-known path. Convention suffices for v1; this is a
discovery contract to design once mechanics are proven.

## IS-6. [FIX][P1] Real node identity once pywaggle runtime calls exist
`nodemeta.py::_runtime_identity()` returns a placeholder today (VSN="NODE",
lat/lon omitted), single swap-in point marked `TODO(sage-ci)`. When the pywaggle/
WES runtime GPS+VSN calls land (Infra #1 / pywaggle2-design §2), wire them in so
filenames carry the real VSN and EXIF carries a real geotag. Plugin-side change
is trivial; the capability is the infra dependency. (Cross-ref: Infra #1.)

## IS-7. [FIX][P1] Cross-user cache read permissions (design 4.2, OPEN)
If an on-node probe shows a consumer pod running as a DIFFERENT user cannot read
the producer's cache files, add world-readable files + traversable dirs
(chmod-on-write) so the producer/consumer story actually works cross-plugin.
This is the Layer-1 (plugin-side) responsibility in the two-layer cache design;
it belongs in the pywaggle2 cache primitive / image-sampler2 cache.py.
Moot until a shared /local-cache mount exists to test against.
(Cross-ref: the `wes-local-cache-manager` repo DESIGN-AND-PURPOSE.md; Infra #9, #4.2.)

================================================================================
bioclip (sage-bioclip)
================================================================================

## BC-1. [FIX][P2] Maintain the pybioclip 2.5 patch (or drop it when upstreamed)
`patch_pybioclip.py` monkey-patches pybioclip at Docker build time to support
BioCLIP 2.5 (adds the 2.5 model to `TOL_MODELS`; makes the text-embedding
filename lookup model-aware). This is our plugin-side workaround for an UPSTREAM
library gap. When pybioclip ships 2.5 support, delete the patch and pin the fixed
version. (Cross-ref: Infra — pybioclip upstream fix, filed under the library-bug
section.)

## BC-2. [FEATURE][P3] Declare resource requirements in sage.yaml
BioCLIP 2.5 Huge (ViT-H/14) OOMKills at `memory=8Gi,limit.memory=16Gi`; needs
`memory=16Gi,limit.memory=32Gi`. Bake correct resource requests into the plugin's
sage.yaml / job YAML so deployments don't OOM opaquely. (Cross-ref: Infra —
per-model resource declaration / clearer failure, platform side.)

================================================================================
Cross-plugin (image-sampler2 + bioclip + yolo + birdnet)
================================================================================

## XP-1. [FEATURE][P2] GPU time-share self-exit (plugin side of the primitive)
Single-GPU nodes can't co-run two always-on GPU plugins. Our in-plugin workaround
is `--max-runtime N`: in continuous mode, loop then self-exit after N seconds so
the plugin behaves like a long bounded single-shot, staggered via cron guard-
bands. image-sampler2 already has this (Stage 3.3, `--max-count`/`--max-runtime`).
Ensure yolo/bioclip/birdnet expose the same bounded-exit so they can be staggered
on one GPU. GOTCHA to handle per plugin: the timer is wall-clock, so a big model
load eats into the window. The DURABLE fix is a scheduler-level GPU lease
primitive (Infra), which would let us retire the per-plugin timers. (Cross-ref:
Infra — GPU time-sharing scheduler primitive.)

## XP-2. [FEATURE][P2] Effective-config startup self-report
Plugins have "soft" config that can fail open silently (geo-filter off, threshold
default, camera-auth wrong) without crashing — invisible until bad data shows up
days later (the birdnet `lon > -1` sentinel bug). Each plugin should publish an
`env.plugin.config` record at startup with its EFFECTIVE config (geo_filter=on/off,
n_species, thresholds, source, camera acquisition_path) so operators verify
engagement from the data plane, not just "pod is Running." The convention/helper
belongs in pywaggle (Infra); emitting it is per-plugin (here). (Cross-ref: Infra
— observable self-checks convention.)

## XP-3. [FIX][P3] Negative-safe sentinels (audit all plugins)
Root cause of the birdnet geo-filter bug: `value > -1` used to mean "is set,"
which is false for all Western-Hemisphere longitudes. Audit every plugin for
sentinel comparisons on values that are legitimately negative (lat/lon,
temperature) and use explicit `== sentinel` / `is None` tests. birdnet is fixed;
verify yolo/bioclip/image-sampler2 have no similar gate.

## XP-4. [FEATURE][P3] Credentials env-only, never in argv (audit all plugins)
Camera credentials must come from the environment (`CAMERA_USER`/
`CAMERA_PASSWORD` or a WES secret) and be redacted in logs — NEVER passed in the
URL/argv (leaks to process listings, logs, git). image-sampler2 already does this;
sage-yolo/sage-bioclip job YAMLs still embed `--snapshot-url http://user:pass@...`.
Migrate them to env-only. The library-default belongs in pywaggle2 (Infra §3.2);
the per-plugin cleanup is here. (Cross-ref: Infra #10.)
