# Reboot Recovery — hummingcam experimental stack (H00F)

**Purpose:** while the hummingcam producer/consumer cascade and its WES support
pieces are still **side-loaded / manually-shimmed** (not yet published to a
registry or folded into the CI stack), a node reboot drops the experimental parts.
This is the single checklist to bring everything back **without re-deriving the
steps each time.** Delete this doc once the CI team makes these components
reboot-durable defaults.

> Node: **H00F** (Thor ARM64). Access: `ssh beckman@node-H00F.sage` (passwordless
> sudo). All `kubectl` = `sudo k3s kubectl`.

---

## Why a reboot breaks it (the mental model)

A reboot clears k3s's containerd image store and kills the running pods. Anything
**published to a registry** re-pulls and recovers on its own; anything
**side-loaded** (imported straight into containerd) or **manually patched into a
live resource** does NOT — it must be re-added by hand.

| Component | Survives reboot? | Why |
|---|---|---|
| Standard WES services (rabbitmq, upload-agent, scoreboard, gps, …) | ✅ auto | registry images, managed by WES |
| `wes-local-cache-manager` DaemonSet | ✅ auto | already a managed DaemonSet on this node |
| `/media/plugin-data/local-cache` (the shared cache dir + data) | ✅ | on-disk, host path — survives |
| **wes-nodeinfo-injection** (ConfigMap + patched scheduler) | ❌ **re-shim** | mutates live resources; patched scheduler is side-loaded |
| **hummingcam-producer** (media-sampler3, image) | ❌ **re-run** | side-loaded image, `pluginctl run` pod |
| **hummingcam-audio-producer** (media-sampler3, audio) | ❌ **re-run** | side-loaded image, `pluginctl run` pod |
| **sage-yolo2-consumer** | ❌ **re-run** | (image persists in containerd if not GC'd, but the pod is gone) |
| **sage-bioclip2-consumer** | ❌ **re-run** | same |

**Detection:** the tell that a reboot happened is every plugin pod showing
`Unknown`/`Failed` and the cache timestamps frozen. Confirm with `uptime` on the
node — a low uptime + stale pods == reboot. (Do NOT assume the control plane
"died"; the pods died, k3s just still lists the stale entries.)

---

## Recovery checklist (run in order)

### 0. Confirm it was a reboot & clear stale pods
```bash
ssh beckman@node-H00F.sage
uptime                                    # low uptime => reboot happened
sudo k3s kubectl get pods -A | grep -iE 'hummingcam|yolo2|bioclip2'   # expect Unknown/Failed
sudo k3s kubectl delete pod hummingcam-producer hummingcam-audio-producer \
    sage-yolo2-consumer sage-bioclip2-consumer \
    --ignore-not-found --grace-period=0 --force
```

### 1. Verify the pieces that should have survived
```bash
sudo k3s kubectl get daemonset wes-local-cache-manager        # 1/1 desired/ready
ls -ld /media/plugin-data/local-cache                         # exists, drwxrwxrwt
sudo k3s ctr images ls | grep -E 'media-sampler3|sage-yolo2|sage-bioclip2'
```
If a side-loaded image is **missing** from containerd (a hard reboot can clear
it), rebuild+re-import it (media-sampler3: native aarch64 `podman build` +
`podman save | sudo k3s ctr images import -`; yolo2/bioclip2 are large — rebuild
from their repos' `scripts/deploy-sideload.sh` if gone). Usually they persist.

### 2. Re-shim wes-nodeinfo-injection (both tiers)
Repo on the node: `~/AI-projects/wes-nodeinfo-injection` (public:
`github.com/flint-pete/wes-nodeinfo-injection`). `git clone` it if absent.

```bash
cd ~/AI-projects/wes-nodeinfo-injection/node-test

# Tier 1 — regenerate the wes-identity ConfigMap with the 5 node vars
./test-add-configmap.sh
# expect: pywaggle2 NodeInfo prints real vsn=H00F + lat/lon, vsn_is_placeholder=false

# Tier 2 — build+side-load the patched edge-scheduler (auto-injects envFrom)
# prereq: populate .upstream/edge-scheduler at the patched base, once per clone:
mkdir -p ../.upstream && \
  git clone https://github.com/waggle-sensor/edge-scheduler.git ../.upstream/edge-scheduler
git -C ../.upstream/edge-scheduler checkout 5391a00
git -C ../.upstream/edge-scheduler apply ../patches/0002-*.patch
./test-add-scheduler.sh          # builds (~3-6 min), rolls out; auto-reverts on failure
```
Tier 2 is optional for THIS stack (media-sampler3 passes `--vsn H00F` explicitly,
and `pluginctl run` pods are built client-side so they don't get the auto-inject
anyway). Re-add it to keep fleet-wide daemon-scheduled plugins getting real
identity. Skip if you only need the hummingcam cascade back.

### 3. Recreate the camera creds file (deleted each session for hygiene)
```bash
umask 077
printf 'CAMERA_HOST=<CAMERA_IP>\nCAMERA_PORT=10000\nCAMERA_CHANNEL=0\nCAMERA_USER=<USER>\nCAMERA_PASSWORD=<PASS>\n' \
  > ~/ms3-img-creds.env
chmod 600 ~/ms3-img-creds.env
```
Real values: the hummingbird camera on the node LAN; the image producer uses the
`test` camera account. Pete holds the current credentials — never commit them.

### 4. Start the producer (media-sampler3 — the great-cut-over image producer)
```bash
sudo pluginctl run --name hummingcam-producer --selector zone=core \
  --env-from ~/ms3-img-creds.env \
  -v /media/plugin-data/local-cache:/local-cache \
  localhost/media-sampler3:0.1.0 -- \
  --continuous 10 --media image --stream top_camera --name top \
  --cache-root /local-cache --cache-name hummingcam \
  --cache-max-count 500 --cache-max-mb 500 --heartbeat-secs 60 --vsn H00F
```
Verify: pod 1/1 Running; fresh `<ts>-v2-H00F-top.jpg` frames appearing in
`/media/plugin-data/local-cache/hummingcam/top/` every 10 s.

### 4b. Start the audio producer (media-sampler3 — 15 s FLAC clip once/min)
A SECOND media-sampler3 instance, independent of the image producer, capturing
from the same camera's mic into its own cache subtree. Needs its own creds file
(the audio path uses the `sage` camera account):
```bash
umask 077
printf 'CAMERA_HOST=<CAMERA_IP>\nCAMERA_PORT=10000\nCAMERA_CHANNEL=0\nCAMERA_USER=<USER>\nCAMERA_PASSWORD=<PASS>\n' \
  > ~/ms3-audio-creds.env
chmod 600 ~/ms3-audio-creds.env

sudo pluginctl run --name hummingcam-audio-producer --selector zone=core \
  --env-from ~/ms3-audio-creds.env \
  -v /media/plugin-data/local-cache:/local-cache \
  localhost/media-sampler3:0.1.0 -- \
  --continuous 60 --clip-seconds 15 \
  --media audio --source-type camera_mic \
  --camera-host <CAMERA_IP> --camera-port 10000 \
  --audio-format flac --bandpass-fmax 8000 \
  --stream hummingcam_mic --cache-name hummingcam-audio \
  --cache-max-count 500 --cache-max-mb 2000 --heartbeat-secs 60 --vsn H00F
```
Verify: pod 1/1 Running; a 15 s FLAC + `.json` sidecar appears every 60 s in
`/media/plugin-data/local-cache/hummingcam-audio/hummingcam_mic/`. (No consumer
reads these yet — a future BirdNET consumer is the planned reader.)

### 5. Start the yolo2 consumer (detector + crop producer)
```bash
sudo pluginctl run --name sage-yolo2-consumer --selector zone=core \
  --resource limit.memory=16Gi,request.memory=4Gi \
  -v /media/plugin-data/local-cache:/local-cache \
  -e WAGGLE_JOB_NAME=hummingcam -e WAGGLE_TASK_NAME=sage-yolo2 \
  registry.sagecontinuum.org/beckman/sage-yolo2:2.1.0 -- \
  --source cache --input /local-cache/hummingcam/top \
  --every 5m --all-unseen --max-frames 0 \
  --model yolo11x.pt --conf-thres 0.25 --classes bird \
  --crop-match "bird:0.4" --crop-padding 0.15 --crop-cache-name hummingcam-crops
```

### 6. Start the bioclip2 consumer (species classifier)
```bash
sudo pluginctl run --name sage-bioclip2-consumer --selector zone=core \
  --resource limit.memory=16Gi,request.memory=4Gi \
  -v /media/plugin-data/local-cache:/local-cache \
  -e WAGGLE_JOB_NAME=hummingcam -e WAGGLE_TASK_NAME=sage-bioclip2 \
  registry.sagecontinuum.org/beckman/sage-bioclip2:2.0.0 -- \
  --source cache --input /local-cache/hummingcam-crops/top-crop-0 \
  --every 10m --all-unseen --max-frames 0 --rank Species --min-confidence 0.1
```

### 7. Verify the whole cascade
```bash
sudo k3s kubectl get pods -A | grep -iE 'hummingcam-producer|hummingcam-audio-producer|yolo2-consumer|bioclip2-consumer'
# all four -> Running, restarts=0

# cloud proof: yolo2 publishes env.count.total to Beehive (frame-anchored)
curl -s -X POST https://data.sagecontinuum.org/api/v1/query \
  -H 'Content-Type: application/json' \
  -d '{"start":"-15m","filter":{"vsn":"H00F","name":"env.count.total"}}' | tail -2
```

---

## Notes / gotchas (learned during CUT-OVER A, 2026-07-23)

- **`pluginctl run` builds pods client-side**, so it bypasses the patched
  scheduler daemon — a plugin launched this way will NOT auto-get
  `envFrom: wes-identity`. media-sampler3 sidesteps this by taking `--vsn` on the
  CLI; other plugins that rely on auto-injection must be scheduled via the cloud
  scheduler / sesctl, or pass identity explicitly.
- **Consumers process oldest-unseen first** (`--all-unseen`). After a reboot there
  may be a stale backlog in the cache; the seen-store persists on disk under
  `/local-cache/.state/`, so consumers resume where they left off and won't
  re-process everything. To reprocess only fresh frames, use `--select-every 0
  --max-frames K` (the K newest) with a fresh `--consumer-id`.
- **Image mode = EXIF-only metadata** (no JSON sidecar; that's audio-only). Don't
  expect `*.json` next to the `.jpg` frames.
- **Data on disk survives** — the cache and seen-stores are host paths; only the
  compute (pods) and the mutable WES shims need restoring.
- **The lasting fix** is CI publishing these images to a registry + folding the
  nodeinfo change into the base stack. Until then, this runbook is the recovery
  path. Tracked in `Infra-problems-to-fix.md`.

## Source-of-truth repos
- `media-sampler3` — producer (`github.com/flint-pete/media-sampler3`)
- `sage-yolo2` — detector/crop producer
- `sage-bioclip2` — species classifier
- `wes-nodeinfo-injection` — the identity shim (`github.com/flint-pete/wes-nodeinfo-injection`)
- `wes-local-cache-manager` — the Layer-2 cache quota DaemonSet
