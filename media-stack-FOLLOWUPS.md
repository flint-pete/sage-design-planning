# Media stack: open follow-ups (state as of 2026-10-02, after the H039 reboot test)

## Status 2026-11-10 (evening): H03B is the test node; Hanwha capture works

**Where we are.** All plugin repos are pushed. media-sampler3 0.3.0 adds Hanwha support
(`--camera-vendor hanwha`) and is verified live on H03B.

- **Why H03B.** H019 is clean and upstream-ready but has no camera. H01B's Hanwha
  XNP-6400RW is in its manifest but answers nowhere on `lan0` (probably unpowered or
  disconnected; needs someone on site). H03B has a live XNP-6400RW at
  `10.31.81.32` (DHCP), plus a Raspberry Pi at `.66`.
- **H03B node state:** WES 697e338, pluginctl and scheduler 0.29.2, `wes-identity`
  GPS 42.5369, -72.1727, no shims, `default-runtime: nvidia`, 3.3 TB free.
- **Fixed on H03B:** the camera provisioner was stuck since Sep 30 (Infra #23,
  stale-Job form); the stuck job was deleted and runs complete again. It still
  never finds the camera, because it only scans `.10`-`.20` (Infra #25; addendum
  note for Sean in `~/AI-projects/_ref/`).
- **Credentials:** Pete created `~/ms3-cam-creds.env` (mode 600) on H03B. It
  currently holds an admin account; a dedicated non-admin viewer account is recommended.

Install progress on H03B (INSTALLING-MEDIA-SAMPLER3.md):
- [x] Step 0: the 5 repos cloned to `~/AI-projects`, at the same commits as GitHub.
- [ ] Step 1: wes-local-cache-manager DaemonSet + `/media/plugin-data/local-cache`.
- [ ] Steps 2–3: identity checks (the read-only check already showed them OK).
- [x] Step 4: `localhost/media-sampler3:0.3.0` in k3s (546.7 MiB).
- [x] Step 5: all built and in k3s, in 27 min total with no cached base image
  (`~/build-consumers.sh`, log `~/build-consumers.log`): sage-yolo2 2.2.0
  (10.9 GiB, 9 min), sage-bioclip2 2.2.0 (17.7 GiB, 12 min), sage-birdnet2 2.1.0
  (2.8 GiB, 6 min). Disk: 154G used of 3.6T.
- [x] Live Hanwha check: a one-shot capture (317,867 B, 1.7 s grab) uploaded, and
  the node's upload agent reported it done. A 2-frame continuous run into the ring
  (scratch dir `/tmp/hanwha-check`) gave native 1920×1080 JPEGs with EXIF GPS =
  `wes-identity`. The night IR image looks fine.
- [ ] Confirm the upload record in Beehive (`location_source`, GPS meta). Blocked:
  the Sage data API returns 500 (`beehive-influxdb ... connection refused`).
- [ ] Steps 6a–6g: producer smoke test, then the seeded bird image and song through
  yolo2, bioclip2 and birdnet2.
- [ ] Step 6h: a live Hanwha producer (`--name ptz`) feeding yolo2
  (`--input /local-cache/camera/ptz`) and bioclip2.
- [ ] Reboot test with REBOOT-RECOVERY.md.
- **Audio, live end to end:** not possible on H03B or H019 (no mic). Needs a node
  with a Reolink mic or a USB mic.

## Update 2026-11-10: identity merged upstream; next test node is H019

- **Node identity is upstream** (WES 697e338, SES/pluginctl 0.29.2, pywaggle 0.57.1).
  Plugins moved to `pywaggle==0.57.1` and stock `pluginctl run`: media-sampler3
  0.2.0, image-sampler2 0.6.0, sage-yolo2 2.2.0, sage-bioclip2 2.2.0, sage-birdnet2
  2.1.0. wes-nodeinfo-injection and pywaggle2-nodeinfo are archived. The Tier
  1/1b/2 items below are history.
- **wes-local-cache-manager is the one platform piece not yet merged.** Keep it
  side-loaded (a WES update does not prune it).
- **H00F's cameras were retasked; H019 is the new test node.** Plan there:
  - [ ] Read-only check: WES >= 697e338 (`wes-identity` has GPS keys), `pluginctl
    version` >= 0.29.2, no leftover `pluginctl-nodeinfo`.
  - [ ] Full install guide run (Steps 0–6h), live camera + mic, all on the
    upstream identity path; confirm `location_source: node` and EXIF GPS.
  - [ ] Reboot test with the revised REBOOT-RECOVERY.md.
- **Retire on older test nodes when updated** (H039, H041, H00F): delete
  `/usr/local/bin/pluginctl-nodeinfo`; let `update-stack.sh` regenerate
  `wes-identity`; remove the `localhost/edge-scheduler:nodeinfo-test` image.
- ~~Then push the five plugin repos~~ Done 2026-11-10 (Pete asked to push before node testing).


Working list for finishing the media-sampler3 stack on Thor nodes. This is a
planning note for the maintainers, not student instructions. The student entry
point is
[INSTALLING-MEDIA-SAMPLER3.md](https://github.com/flint-pete/media-sampler3/blob/master/INSTALLING-MEDIA-SAMPLER3.md).

## Where things stand

**Platform changes (side-loaded; not yet in the WES base image):**
- **wes-local-cache-manager:** bounds `/local-cache`.
- **wes-nodeinfo-injection:**
  - Tier 1: the `wes-identity` ConfigMap with 5 identity vars.
  - Tier 1b: `pluginctl-nodeinfo`, a patched `pluginctl` whose pods get those vars.
  - Tier 2: an optional patched scheduler, for SES jobs.
- **pywaggle2-nodeinfo:** the reader library, vendored into the plugins.

**Producer:** media-sampler3 (images and audio).

**Example consumers:**
- sage-yolo2 (detect and crop)
- sage-bioclip2 (species from crops)
- sage-birdnet2 (BirdNET on audio clips)

**H039 fresh-install run (Oct 2026, no cameras attached).** The guide's Steps 0–5
and 6a–6g all passed:
- **Producer check** against a fake camera address passed.
- **Seeded cardinal image:** yolo2 published `env.count.bird`, and bioclip2
  published *Cardinalis cardinalis*.
- **Seeded bluebird clip:** birdnet2 published *Sialia sialis*, using the node's
  GPS for the eBird filter.
- **Beehive:** all records arrived with H039's VSN and lat/lon.

Run log: `~/.hermes/cache/scratch/h039-run.md` on Flint (local, not in git).

**Current state of H039:**
- **Running:** `sage-yolo2-consumer`, `sage-bioclip2-consumer` and
  `sage-birdnet2-consumer`, all launched with `pluginctl-nodeinfo`. The cache
  directories they read are empty.
- **Installed:**
  - `/usr/local/bin/pluginctl-nodeinfo`
  - Tier 1 ConfigMap (5 vars)
  - cache manager
  - all four images
- **Stock scheduler** `waggle/edge-scheduler:0.28.0` (Tier 2 was removed after the
  reboot test). The root-podman image `localhost/edge-scheduler:nodeinfo-test` is
  kept, because `install-pluginctl-nodeinfo.sh` reuses it.
- **Seeds removed and probe seen-store deleted.** The seeded crops remain in
  `camera-crops/top-crop-0` (the cache manager bounds them).

---

## 1. Tomorrow, with hardware

- [ ] **Attach the camera and microphone; run install guide Step 6h.**
  - Start the image producer and the audio producer with `pluginctl-nodeinfo`, with
    no identity flags.
  - Confirm the images and clips appear, and `env.count.total`,
    `env.detection.audio.summary` and heartbeats reach Beehive.
  - Check both producers' frames and sidecars carry H039's GPS.
  - This is the first live run of the audio producer outside H00F, and the first
    live microphone → birdnet2 run anywhere.
- [x] **Reboot test, done 2026-10-02 (H039, no camera).**
  - Followed `REBOOT-RECOVERY.md` literally. Everything survived: images, the
    5-var ConfigMap, `pluginctl-nodeinfo`, the cache manager, `/local-cache` and
    the seen-stores, and the Tier 2 scheduler.
  - The `pluginctl` pods vanished, and the docs now say so. Step 0's grep was
    narrowed.
  - Step 8 relaunch, then the seeded test: all three consumers reached Beehive
    with lat/lon.
  - Added a step-9 no-camera check (bioclip2 needs new pixels: a mirrored seed).
- [ ] **Re-run REBOOT-RECOVERY steps 5–7 after 6h:** camera credentials,
  reachability, and both producers. These were skipped because no camera is
  attached. A quick second reboot with the producers running would cover them.
- [x] **Tier 2 removed from H039** (`test-remove-scheduler.sh`, stock 0.28.0
  restored, pluginctl pods untouched). Its envFrom injection remains verified only
  on H00F (it needs a cloud SES job).
- [x] **Clean up H039:** probe seen-store deleted.

## 2. Release tags: DONE 2026-10-02 (before the live camera test, by decision)

Tagged now. Any fixes the live camera test needs ship as patch releases.

- [x] media-sampler3 **v0.1.1** (image `localhost/media-sampler3:0.1.1`; producer
      code unchanged from 0.1.0)
- [x] wes-nodeinfo-injection **v1.1.0** (Tier 1b `install-pluginctl-nodeinfo.sh`)
- [x] pywaggle2-nodeinfo **v0.1.2** (docs only; the vendored copies stay at v0.1.1)
- [x] sage-yolo2 **v2.1.1** (pinned dependencies, opencv fix)
- [x] sage-bioclip2 **v2.1.0** (GPU via `--device auto`, offline model loading,
      pinned dependencies)
- [x] sage-birdnet2 **v2.0.1** (first tag: `birdnet==0.2.16` pin plus all
      dependencies pinned)
- wes-local-cache-manager: unchanged since **v0.2.1**, so no new tag.
- sage-design-planning: planning notes, not tagged.

## 3. Security

- [ ] **Rotate the Reolink camera password.** The old password is still in
      media-sampler3's git history and in the public v1 `flint-pete/sage-yolo`
      job YAMLs. Rotating it on the cameras is the fix; rewriting public history
      doesn't help once it has been cloned.
- [ ] **Credentials are visible in the pod spec.** `pluginctl --env-from` copies
      `CAMERA_USER`/`CAMERA_PASSWORD` into the pod spec as plain env values
      (visible with `kubectl get pod -o yaml` in `default`). Scheduled jobs should
      use a Kubernetes Secret (`envFrom: secretRef`). The install guide documents
      this.

## 4. Latent build and runtime risks (small fixes)

- [x] **Pinned Python dependencies (done 2026-10-02).** sage-yolo2, sage-bioclip2
      and sage-birdnet2 pin every direct dependency to the H039-verified
      versions. This includes bioclip2's open_clip, huggingface_hub and timm, and
      birdnet2's tensorflow. Also fixed the unquoted
      `opencv-python-headless>=4.8.0` in the yolo2/bioclip2 Dockerfiles: the
      shell read it as a redirect, so pip installed the newest opencv. All three
      images rebuilt and passed the seeded test on H039.
- [x] **bioclip2 offline (done 2026-10-02).** `ENV HF_HUB_OFFLINE=1`, documented as
      a "Sage adjustment" in its README.
  - Before: 5 HEAD requests per start, resolving `main`, so the model could change
    without notice.
  - After: 0 requests.
  - Works under `docker run --network none`. The old image did too (it falls back
    to the cache when the network refuses the connection), so the old "needs
    outbound network" claim was wrong. The real risks were the model changing
    and stalls on a network that silently drops traffic.
  - yolo2 was also confirmed to run with no network.

## 5. For the Sage CI team (track and hand off)

- [x] **GPU: RESOLVED by the CI team (2026-10-02).** They set k3s
  `default-runtime: nvidia` across the fleet.
  - **Config:** `/etc/rancher/k3s/config.yaml` becomes containerd
    `default_runtime_name = "nvidia"`, confirmed on H039, H041 and H00F. It's k3s's
    own setting, so it survives k3s upgrades.
  - **Verified on H039:** the sage-yolo2 pod has `/dev/nvidia*` and
    `torch.cuda.is_available()` = True (NVIDIA Thor), and logs
    `Loading yolo11x.pt on cuda`. A throwaway pod ran BioCLIP-2.5 on CUDA:
    0.12 s/crop vs 1.86 s on the CPU, same result.
  - **Background:** before the fix, H01A/H038/H039/H041/H043 defaulted to `runc`,
    and only H00F had a hand-made `config.toml.tmpl` pointing runc at
    `nvidia-container-runtime`. Ask the CI team whether that template can now be
    removed from H00F, since it freezes H00F's containerd config across upgrades.
  - **sage-bioclip2 fixed too (our code):** it now passes `device` to pybioclip
    (`--device auto`) and logs `on cuda` on H039.
- [ ] **Fold the node-identity change into WES:** patch 0001 (ConfigMap generator)
      and patch 0002 (pod builder). Ship 0002 in **both** the scheduler and the
      host `pluginctl` binary. Otherwise `pluginctl` pods stay unpatched, which is
      why Tier 1b exists.
- [ ] **Publish the images** (media-sampler3, sage-yolo2, sage-bioclip2,
      sage-birdnet2, plus the cache manager) through ECR, now that it builds Thor
      images, and test the SES job templates in `media-sampler3/jobs/`. Until then
      the stack is side-loaded and needs `REBOOT-RECOVERY.md`.
- [ ] Update this repo's existing infra notes:
  - `Infra-problems-to-fix.md` should say the `pluginctl` client-side issue has a
    side-load fix, Tier 1b;
  - add the GPU item above.

## 6. Known limitations: good student starter tasks (already documented in the READMEs)

- [x] sage-bioclip2: pass a `device` to `TreeOfLifeClassifier` (done 2026-10-02:
      `--device auto`, on cuda on H039).
- [ ] sage-bioclip2: read every `*-crop-N` directory, not just `top-crop-0`.
- [ ] Hand-seeded test files (no `unique_id`) are reprocessed on every wake: mark
      them seen by path or content hash.
- [ ] All three consumers keep seen-stores under `.state/sage-yolo2/...`, a quirk
      of the shared consumer code. Make the prefix the consumer's own name, with
      migration.
- [ ] Legacy `IS2_*` env names (from image-sampler2) are still used for
      compatibility.
- [ ] `media-sampler3/jobs/*.yaml` SES templates are untested.

## 7. Done (2026-10-01/02), for context

- Student-readiness pass across all repos: current docs vs `docs/history/`,
  HOW-IT-WORKS, DESIGN-PATH, REBOOT-RECOVERY, secrets scrubbed from current files.
- Removed the "ECR can't build Thor images" notes; the CI team fixed that.
- Tier 1b (`pluginctl-nodeinfo`) built, verified and made the single launch path.
- Install guide simplified to one path: one camera, no identity flags, Tier 2
  moved out to wes-nodeinfo-injection's docs.
- sage-birdnet2 integrated as the third consumer: `birdnet==0.2.16` pin, seeded
  bluebird test, 2Gi memory limit. The deprecated `birdnet` repo is no longer
  referenced.
- Reboot test on H039 and Tier 2 removal (2026-10-02, details in section 1).
- Fixes found on H039: `make test` bootstrap, `--stream` documented as required,
  producer exit-2 table, Tier 2 verification (`-n ses`, needs SES), seed removal
  without wildcards.
