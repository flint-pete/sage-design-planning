# Sage Cyberinfrastructure Issues Report
## From Edge Plugin Development: YOLO + BioCLIP on Thor H00F
### Pete Beckman — June 18, 2026

> **SUPERSEDED (2026-07-07):** This is the ORIGINAL dated report. Its live items
> have been folded into `Infra-problems-to-fix.md` (the authoritative
> infra tracker) — issue mapping: #1 NRP storage 404 -> Infra #18; #2 ECR arm64
> QEMU crash -> Infra #3; #3 pybioclip 2.5 -> Infra #19 (+ plugin-improvements
> BC-1); #4 BioCLIP OOM -> Infra #20 (+ plugin-improvements BC-2); #5 SSH key loss
> -> Infra #21. RETAINED as a historical snapshot for its verification detail
> (NRP 404 curl commands, exact QEMU SIGABRT trace, pybioclip root-cause analysis,
> environment table). Do not add new items here — use Infra-problems-to-fix.md.

---

## Summary

During development and deployment of YOLO and BioCLIP edge plugins on
Sage node H00F (Thor, 128GB unified memory, arm64), we encountered five
infrastructure issues that affect plugin developers and the broader Sage
ecosystem.  Issues are ranked by severity.

---

## 1. NRP Storage Not Receiving Uploads (Critical)

**Status:** Broken globally — affects all nodes

**Symptom:** Plugin images uploaded via pywaggle's `plugin.upload_file()`
appear as records in the Sage data API (`data.sagecontinuum.org`) with
valid storage URLs, but the actual files return HTTP 404 from NRP storage
(`nrdstor.nationalresearchplatform.org`).

**Scope:** Not limited to our plugins or node.  We verified:
- YOLO annotated images from H00F (`Pluginctl/sage-yolo-hummingcam-0.2.0`) → 404
- plebbyd's BioCLIP uploads from H00F (`bioclip2-hummingbird-5640`) → 404
- Weather station uploads from W08D (`sage-waggle-wxt536-1.00`) → 404
- Old files uploaded before the outage (~04:55 UTC June 18) still work

**Where the pipeline is healthy:**
- The upload agent on Thor (`wes-upload-agent`) works correctly — files are
  rsync'd to `beehive-uploads.sagecontinuum.org` (165.124.33.154) and cleaned
  up immediately.  No backlog, no errors in upload agent logs.
- The data API receives and serves metadata records with correct timestamps
  and storage URLs.

**Where it breaks:**
- Files reach Beehive but do not propagate to NRP storage replicas.
- The NRP endpoint (`nrdstor.nationalresearchplatform.org`, resolving to
  `hcc-nrdstor-osdf.unl.edu` / `129.93.153.195`) returns a text error:
  `Unable to open /sage/node-data/...; no such file or directory`

**Impact:** Any plugin that uploads annotated images or data files is
silently broken.  Text/numeric measurements still flow correctly through
the data API — only file uploads are affected.

**Verification commands:**
```bash
# This old upload works:
curl -L -u USER:TOKEN -o test.jpg \
  "https://storage.sagecontinuum.org/api/v1/data/Pluginctl/sage-yolo-hummingcam-0.2.0/00004cbb4701d16c/1781758539101986418-http-snapshot-annotated.jpg"
# Returns: 25KB JPEG

# Any recent upload fails:
curl -L -u USER:TOKEN -o test.jpg \
  "https://storage.sagecontinuum.org/api/v1/data/Pluginctl/sage-yolo-hummingcam-0.2.0/00004cbb4701d16c/1781783229481847410-http-snapshot-annotated.jpg"
# Returns: HTTP 404, 160 bytes ASCII text
```

---

## 2. ECR Multi-Arch Build Fails for GPU Plugin Images

**Status:** Blocking ECR submission for arm64 GPU plugins

**Symptom:** The ECR Jenkins pipeline builds both `linux/amd64` and
`linux/arm64` from the same Dockerfile.  The amd64 build succeeds.
The arm64 build crashes during the first `RUN` step that imports PyTorch:

```
#17 [linux/arm64  3/10] RUN TORCH_VER=$(python3 -c "import torch; ...")
#17 234.5 qemu: uncaught target signal 6 (Aborted) - core dumped
```

**Root cause:** The ECR builder runs on amd64 hardware and uses QEMU
user-mode emulation for arm64 builds.  The NVIDIA PyTorch base image
(`nvcr.io/nvidia/pytorch:25.08-py3`) contains GPU-specific shared
libraries (CUDA, cuDNN, NCCL) that crash under QEMU emulation when
Python imports them.

**Impact:** Any plugin using the NVIDIA PyTorch base image cannot be
built for arm64 on the current ECR infrastructure.  This includes
YOLO, BioCLIP, vLLM, and any future GPU-accelerated plugin.

**Possible solutions:**
1. Provide a native arm64 builder node for ECR (e.g., a Sage Thor or
   DGX Spark node enrolled as a Jenkins agent)
2. Allow plugins to specify single-architecture builds in sage.yaml
   (e.g., `architectures: ["linux/amd64"]` only)
3. Use a cross-compilation strategy that doesn't require running
   target-arch binaries (difficult with PyTorch/CUDA)

**Note:** Our YOLO plugin (sage-yolo) passed ECR — we need to verify
whether it also lists both architectures or only amd64.

---

## 3. pybioclip Library Does Not Support BioCLIP 2.5

**Status:** Workaround in place (monkey-patch); upstream fix needed

**Symptom:** `pybioclip` 2.1.5 (latest on PyPI as of June 2026) raises:
```
ValueError: TreeOfLife predictions are only supported for the
following models: hf-hub:imageomics/bioclip, hf-hub:imageomics/bioclip-2
```
when passed `model_str='hf-hub:imageomics/bioclip-2.5-vith14'`.

**Root cause:** Two issues in pybioclip:

1. `_constants.py` — `TOL_MODELS` dict only contains entries for `bioclip`
   and `bioclip-2`.  BioCLIP 2.5 maps to the same dataset repo
   (`imageomics/TreeOfLife-200M`) but is not listed.

2. `predict.py` — `get_txt_emb()` and `get_txt_names()` hardcode the
   filenames `txt_emb_species.npy` and `txt_emb_species.json`.  BioCLIP 2.5
   uses model-specific filenames: `txt_emb_bioclip-2.5-vith14.npy` and
   `txt_emb_bioclip-2.5-vith14.json`.  These files exist in the
   TreeOfLife-200M dataset repo under `embeddings/` but pybioclip doesn't
   know to look for them.

**Our workaround:** A `patch_pybioclip.py` script applied at Docker build
time that:
- Adds the 2.5 model to `TOL_MODELS` in `_constants.py`
- Adds a `TOL_EMB_FILES` dict mapping model strings to their specific
  embedding filenames
- Patches `get_txt_emb()` and `get_txt_names()` in `predict.py` to use
  model-specific filenames with fallback to the defaults

**Recommended upstream fix:** Add BioCLIP 2.5 to `TOL_MODELS` and make
the embedding filename lookup model-aware.  The text embeddings already
exist in the dataset repo.  This is a ~15-line change to `_constants.py`
and `predict.py`.

**Contact:** pybioclip maintainers: egrace479, jbradley
(github.com/Imageomics/pybioclip)

---

## 4. BioCLIP 2.5 OOMs at Standard Resource Limits

**Status:** Resolved with increased limits; documentation needed

**Symptom:** BioCLIP 2.5 Huge (ViT-H/14) deployed via pluginctl with
`--resource "memory=8Gi,limit.memory=16Gi"` gets OOMKilled during
initialization:

```
$ sudo kubectl get pods | grep bioclip
bioclip-hummingcam   0/1   OOMKilled   0   2m27s
```

**Root cause:** BioCLIP 2.5 uses a ViT-H/14 backbone (much larger than
BioCLIP 2's ViT-L/14) and its TreeOfLife text embeddings are 3.0 GB
(vs 2.5 GB for BioCLIP 2).  During initialization, the model weights
(~4-5 GB) plus the text embeddings (~3 GB) plus PyTorch overhead exceed
the 16 Gi memory limit.

**Resolution:** Increasing to `memory=16Gi,limit.memory=32Gi` resolves
the issue.  Both values fit comfortably in 128 GB unified memory nodes.

**Recommendation:** pluginctl or the ECR portal should document per-model
resource requirements, or provide a way for plugins to declare minimum
resource requirements in sage.yaml so that insufficient deployments fail
with a clear message rather than an opaque OOMKill.

---

## 5. SSH Agent Key Loss on Long Operations via sage-vpn

**Status:** Partially mitigated; root cause unclear

**Symptom:** SSH connections to `*.sage` nodes via the `sage-vpn` proxy
fail with `Permission denied (publickey,password)` after extended
operations (5-15 minutes).  This happens repeatedly during Docker builds,
k3s image imports, and other long-running tasks that rely on polling
the node over SSH.

**Diagnosis:** The original `~/.ssh/config` had:
- `sage-vpn` host: `ControlMaster auto`, `ControlPersist 10m`,
  `IdentityFile ~/.ssh/sage_key`
- `*.sage` host: `ProxyJump sage-vpn` but NO `IdentityFile` — relied
  entirely on the SSH agent to provide the key

When the SSH agent's key expired or the control master connection
timed out, new connections through the proxy couldn't authenticate
because the `*.sage` block had no key file configured.

**Mitigation applied:**
```
Host *.sage
    ProxyJump sage-vpn
    User root
    IdentityFile ~/.ssh/sage_key
    IdentitiesOnly yes
```

This helps but doesn't fully resolve the issue — the agent key still
expires periodically, requiring manual `ssh-add`.

**Workaround:** All long-running operations on Thor are launched via
`nohup` so they survive SSH disconnections.  Polling loops check for
completion but are resilient to SSH failures.

---

## Environment Details

| Component | Value |
|-----------|-------|
| Node | H00F (Thor, sgt-thor-1423225065558-H00F) |
| Node MAC | 00004cbb4701d16c |
| Hardware | NVIDIA Thor, sm_110, 128GB unified memory, aarch64 |
| Base image | nvcr.io/nvidia/pytorch:25.08-py3 |
| YOLO plugin | yolo-object-counter v0.2.0 (yolo11x.pt) |
| BioCLIP plugin | bioclip-species-classifier v0.3.0 (BioCLIP 2.5 ViT-H/14) |
| pybioclip | 2.1.5 (PyPI) |
| Camera | Reolink on LTE via port-mapped router |
| Development host | DGX Spark (Flint), GB10 sm_121, 128GB unified |

---

*Report prepared during hummingbird feeder monitoring project,
deploying YOLO + BioCLIP edge plugins on Thor H00F.*
