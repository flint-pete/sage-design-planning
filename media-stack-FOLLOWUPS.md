# Media stack: open follow-ups (state as of 2026-10-02)

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
  - **Tier 2 patched scheduler** (`edge-scheduler:nodeinfo-test`)
  - cache manager
  - all four images
- **Leftover:** `/local-cache/.state/sage-yolo2/probe-gps/` (a probe's seen-store;
  safe to delete).

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
- [ ] **Do a real reboot test using `REBOOT-RECOVERY.md`,** right after 6h works,
  so the whole stack is covered. Follow it literally and fix anything that's
  wrong or unclear. Check in particular:
  - side-loaded images survive;
  - `wes-identity` keeps its 5 vars;
  - `/usr/local/bin/pluginctl-nodeinfo` is still there;
  - the Tier 2 scheduler pod comes back (it has never been tested across a
    reboot).
  - H039 looks shared (about 60 user namespaces), so warn other users first.
- [ ] **Remove Tier 2 from H039 after the reboot test**
  (`wes-nodeinfo-injection/node-test/test-remove-scheduler.sh`). The guide no
  longer uses it. Its envFrom injection is verified only on H00F, because it needs
  a cloud SES job. If you want it verified on H039 first, submit one SES job to
  H039 before removing it.
- [ ] **Clean up H039:** delete `/local-cache/.state/sage-yolo2/probe-gps/`.

## 2. Release tags (after the reboot test, so the tags match what was verified)

- [ ] media-sampler3: **v0.1.1** (6 commits since v0.1.0: single-path guide,
      birdnet2 integration, `make test` bootstrap, GPU caveat).
- [ ] wes-nodeinfo-injection: **v1.1.0** (Tier 1b `install-pluginctl-nodeinfo.sh`,
      shared build helper, `/usr/local/bin` default).
- [ ] sage-birdnet2: bump `sage.yaml` to **2.0.1** (the dependencies changed:
      `birdnet==0.2.16` pin) and tag `v2.0.1`; it has never been tagged.
- [ ] Decide whether sage-yolo2, sage-bioclip2 and pywaggle2-nodeinfo need tags.
      Their changes since the last tag are docs only.

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

- [ ] **Pin Python dependencies to what built on H039.** The birdnet2 break came
      from an open-ended `birdnet>=0.2.16`, which picked up a breaking 1.x.
      Still open-ended:
  - sage-yolo2: `ultralytics>=8.3.70`, `pywaggle[all]>=0.56.0`,
    `opencv-python-headless>=4.8.0`, `pillow>=10.0.0`
  - sage-bioclip2: `pywaggle[all]`, `opencv`, `pillow`, `numpy` (pybioclip is
    already pinned)
  - sage-birdnet2: `librosa>=0.11`, `numpy`, `soundfile`, `pywaggle[audio]`

  To get the working set: `pip freeze` inside each running pod on H039, then pin
  those versions (or add them to each repo's existing constraints file).
- [ ] **bioclip2 contacts huggingface.co at startup** (HEAD requests), even though
      the weights are baked in. Test with outbound network blocked, or set
      `HF_HUB_OFFLINE=1` in the image, so it works on nodes without internet.

## 5. For the Sage CI team (track and hand off)

- [ ] **GPU: no plugin gets the GPU on Thor nodes other than H00F.** File this in
      `Infra-problems-to-fix.md`. Evidence and options:
  - **Survey (Oct 2026):** H01A, H038, H039, H041 and H043 default to plain
    `runc`. No `default_runtime_name`, no k3s `default-runtime`, no
    `nvidia.com/gpu` resource, no device plugin. NVIDIA's runtime is installed and
    registered on all of them.
  - **Inside running pods on H039 and H041:** runtime handler is the default, no
    `/dev/nv*`, `torch.cuda.is_available()` = False, and yolo2 logs
    `Loading yolo11x.pt on cpu`. This happens even though the CUDA image sets
    `NVIDIA_VISIBLE_DEVICES=all`.
  - **H00F is the exception:** a hand-made
    `/var/lib/rancher/k3s/agent/etc/containerd/config.toml.tmpl` (2026-04-03) sets
    `runtimes.runc.options BinaryName = "/usr/bin/nvidia-container-runtime"`. Note
    that this freezes H00F's containerd config across k3s upgrades.
  - **Code:** edge-scheduler (latest, 5391a00) and `pluginctl` never set
    `runtimeClassName`. `limit.gpu` maps to `nvidia.com/gpu`, which no node
    advertises.
  - **Options:**
    - (a) make NVIDIA's runtime the default (k3s `default-runtime: nvidia`, or
      H00F's template);
    - (b) have the scheduler and `pluginctl` set `runtimeClassName: nvidia` for
      GPU plugins;
    - (c) add a GPU device plugin so the scheduler can account for GPU use.
  - **Separately:** sage-bioclip2 never asks pybioclip for a device, so it runs on
    the CPU even on H00F (a code fix, below).
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

- [ ] sage-bioclip2: pass a `device` to `TreeOfLifeClassifier` (always CPU today).
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
- Fixes found on H039: `make test` bootstrap, `--stream` documented as required,
  producer exit-2 table, Tier 2 verification (`-n ses`, needs SES), seed removal
  without wildcards.
