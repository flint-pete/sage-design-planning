# Sage Cyberinfrastructure — Potential Features & Interface Improvements

> **SUPERSEDED (2026-07-07):** Items from this draft have been sorted into the two
> authoritative trackers by the infra-vs-plugin axis:
> - INFRA (platform, outside plugins) -> `Infra-problems-to-fix.md`:
>   §1 GPS/location -> Infra #1; §2 ECR arm64 -> Infra #3; §3 ECR catalog API ->
>   Infra #14; §4 GPU time-share (primitive) -> Infra #15; §5 provenance stream ->
>   Infra #16; §6 sesctl docs -> Infra #17; §8 smaller (sentinels/imagePullPolicy/
>   camera cookbook/model-load) -> Infra #22.
> - PLUGIN (inside our plugins) -> `plugin-improvements.md`:
>   §4 GPU self-exit (plugin side) -> XP-1; §7 effective-config self-report -> XP-2;
>   §8 sentinel audit -> XP-3; credentials env-only -> XP-4.
> RETAINED as the original narrative draft (fuller prose/rationale per item). Do not
> add new items here — use the two trackers above.

A running list of capability gaps and interface friction discovered while
building and operating real edge-AI plugins (BirdNET audio, YOLO object
counting, BioCLIP species classification) on Sage nodes — primarily the H00F
hummingbird cam (Thor, arm64, single GPU) and related deployments.

Each item states the problem we hit, why it matters beyond our one use case,
and a concrete proposal. Author: Pete Beckman. Status: draft for discussion
with the Sage CI team.

---

## 1. A first-class way for plugins to get their location (GPS / node coordinates)

**Problem.** A plugin that does location-aware science (e.g. BirdNET's eBird
geo-filter, which restricts predictions to species expected at a lat/lon/week)
has no reliable, supported way to learn *where it is running*. We found:

- pywaggle (0.56) has **no location API** — no `waggle.data.gps`, no
  `Plugin.get_location()`. The only live mechanism is subscribing to the
  `sys.gps.*` measurement stream, which only exists on GPS-equipped/mobile nodes.
- SES does **not** mount the node manifest (`node-manifest-v2.json`) into plugin
  pods, so a plugin can't read fixed-node coordinates from the manifest either.
- Net result: for a fixed node, the only working option is to **hard-code
  `--lat`/`--lon` in every job YAML**, per node. That doesn't scale and is
  error-prone (and silently degrades science quality when omitted — see §7).

**Why it matters.** Location is fundamental metadata for almost any
environmental science plugin (geo-filtering, sun position / solar irradiance,
timezone, regional models, data provenance). Every such plugin currently
reinvents this badly.

**Proposal.**
- Add a first-class pywaggle accessor, e.g. `Plugin.get_location()` →
  `{lat, lon, alt, source, timestamp}`, that transparently resolves from the
  best available source (live GPS if present, else node-assigned fixed
  coordinates).
- Have WES inject the node's assigned coordinates into every plugin pod
  (env vars `WAGGLE_NODE_GPS_LAT/_LON/_ALT`, or a mounted, read-only manifest
  fragment) so fixed nodes work without per-job hard-coding.
- Document the precedence clearly (live GPS > injected fixed coords > none).

---

## 2. The ECR build system must natively support Thor / arm64 + NVIDIA base images

**Problem.** Plugins targeting Thor (NVIDIA, arm64, sm_110) cannot be built
through the normal ECR portal pipeline:

- The portal build runs arm64 under **QEMU emulation** and the NVIDIA-base
  image build crashes with SIGABRT (exit 134) during emulated execution.
- Separately, our namespace token **lacks Docker registry push scope**, so we
  cannot push a locally-built arm64 image to the registry as a workaround.

**Current workaround (manual, per-node, not scalable).** Build the image
locally on the Thor node, tag it with the full registry path, **sideload** it
into the node's k3s containerd (`docker save | sudo k3s ctr images import -`),
and separately register only the **catalog metadata** via the ECR API (see §3).
SES pods use `imagePullPolicy=IfNotPresent`, so the sideloaded image serves the
actual pull. This works but must be repeated by hand on every node and every
version.

**Why it matters.** Thor/Spark-class hardware is the future of GPU edge
inference on Sage. Any plugin author targeting it hits this wall. It blocks the
"push to git → portal builds → deploy" story entirely for arm64 GPU plugins.

**Proposal.** Either (or both):
- (a) Add a **native arm64 builder** (real arm64 runners, not QEMU) to the
  Jenkins/ECR build pipeline, so NVIDIA-base arm64 images build like any other.
- (b) Grant scoped **registry push access** per namespace so authors can push a
  locally/natively-built image when the hosted builder can't.

---

## 3. ECR catalog registration as a documented, first-class API (decoupled from the build)

**Problem.** SES validates a job's image against the ECR **app catalog**
(`ecr.sagecontinuum.org`) — not the Docker registry and not the image actually
present on the node. If the catalog has no record for the exact version,
`sesctl submit` fails with `... does not exist in ECR`, *even when the image is
sideloaded and runnable*. Today the only blessed way to create that catalog
record is the portal "Create App / add version" UI, which is tied to the
(broken-for-Thor) build step.

**What we discovered.** Catalog registration **can** be done directly via the
API: `POST https://ecr.sagecontinuum.org/api/submit` with header
`Authorization: Sage <portal-token>` and the full app metadata JSON (we clone an
existing version record via `GET /api/apps/<ns>/<name>/<ver>`, bump the version
+ git source, and re-POST; `description` is a required field). This cleanly
separates "register metadata" from "build image."

**Why it matters.** Decoupling catalog registration from the build unblocks the
sideload path (§2) and any other "image built elsewhere" scenario. It's also
just good architecture — validation metadata shouldn't be welded to one build
mechanism.

**Proposal.** Officially support and document a "register a version" API
(and/or a `sesctl`/CLI subcommand) that registers catalog metadata
independently of building the image. Make `imagePullPolicy=IfNotPresent` +
catalog-only registration a *supported* deployment mode, not an accident.

---

## 4. A bounded-runtime / GPU time-sharing primitive in the scheduler

**Problem.** A node with a **single GPU** cannot run two always-on continuous
GPU plugins — a held GPU blocks the second pod from scheduling at all. SES has
no native "run this plugin for 10 minutes, then yield the GPU" concept.

**Current workaround (in-plugin).** We added a `--max-runtime N` flag to each
plugin so that, in continuous mode, it loops sampling then **self-exits** after
N seconds — behaving like one long bounded single-shot. We then stagger two
plugins with cron + guard-bands (YOLO :00–:10, BioCLIP :20–:30) so they share
one GPU at ~20 min/hour total. This works but pushes scheduling policy *into
every plugin's code*, and the timer is wall-clock (model-load eats into the
window — a subtle gotcha each author must handle).

**Why it matters.** Single-GPU nodes are common. Time-slicing the GPU between
models is a recurring need that shouldn't require every plugin to implement its
own self-exit timer and every operator to hand-tune cron offsets.

**Proposal.**
- A scheduler-level **GPU lease / time-window** primitive: declare a plugin
  should hold the GPU for a bounded window on a cadence, and let SES enforce
  start/stop and mutual exclusion (with guard-bands) across plugins contending
  for `resource.gpu`.
- Alternatively/additionally, a node-level **GPU mutex** so two GPU plugins
  can be co-scheduled safely with the scheduler serializing access.

---

## 5. Versioned provenance / data-quality annotation markers in the data stream

**Problem.** When a plugin bug is found and fixed (e.g. our geo-filter bug —
records before birdnet 0.1.4 contain unfiltered global species), there is no
standard way to **annotate the archive** to say "data in this window, from this
plugin version, has caveat X." We intentionally keep historical data rather than
deleting it, so consumers need a machine-readable way to know what's trustworthy.

**What exists today.** Every record carries the exact image version in its
`meta.plugin` tag (`registry.sagecontinuum.org/<ns>/<name>:<ver>`), which can
serve as a join key — but there's no place to attach the *meaning* (which
versions are known-bad, for what reason, over what time range).

**Why it matters.** Reproducible science requires provenance. Long-lived
archives accumulate behavior changes; without annotations, every downstream
analyst must rediscover them.

**Proposal.**
- A standard **annotation / data-quality stream** (e.g. `env.annotation.*` or a
  dedicated provenance topic) carrying `{start_ts, end_ts, node, plugin,
  version, severity, note}` markers that data clients can join against.
- And/or a **sidecar provenance record per job/version** in ECR or Beehive, plus
  a documented query pattern in `sage_data_client` for "give me only records
  from plugin versions >= X" / "flag records overlapping a known-issue window."

---

## 6. Correct and authoritative `sesctl` documentation

**Problem.** The published `sesctl` docs
(https://sagecontinuum.org/docs/reference-guides/sesctl and the linked
edge-scheduler README) do not match the actual binary:

- Docs imply `sesctl create --from-file`; the binary uses `-f` / `--file-path`.
- Docs imply submit/manage **by job name**; the binary requires
  `submit -j <numeric-job-id>` (the numeric ID returned by `create`). Suspend is
  `rm -s <id>`, remove is `rm <id>`.

**Why it matters.** Every new plugin author follows these docs and hits
immediate, confusing failures. It's a small fix with high friction-reduction.

**Proposal.** Correct the reference docs and the edge-scheduler README to the
real flag names and the create-returns-numeric-id → submit-by-id workflow.
(We've already patched our internal skill; an upstream issue/PR is warranted.)

---

## 7. "Is my filter / config actually engaged?" — observable plugin self-checks

**Problem.** Our geo-filter bug was invisible for a long time because the plugin
*accepted* the `--lat/--lon` args, the pod *logged* the coordinates in its args,
and the job *ran* — but the filter silently never engaged (a sentinel-comparison
bug: `lon > -1` is false for all negative/Western-Hemisphere longitudes). The
only real evidence was out-of-range species appearing in the data days later.

**Why it matters.** Plugins routinely have "soft" configuration that can fail
open (filter off, threshold default, geo disabled) without crashing. Operators
need a way to confirm intended behavior is *actually active*, not just that the
job is "Running."

**Proposal.**
- Encourage/standardize a plugin **startup self-report** published as metadata
  (e.g. `env.plugin.config` with the effective config: geo_filter=on/off,
  n_species, thresholds, source) so operators can verify engagement from the
  data plane, not just pod logs.
- Consider a lightweight **plugin health/“effective config” convention** in
  pywaggle so this is uniform across plugins.

---

## 8. Smaller items / nice-to-haves

- **Negative-value-safe sentinels in shared templates.** Our bug came from
  `value > -1` meaning "is set." Any Sage/pywaggle example or template that
  gates on coordinates/temperatures/etc. should use explicit `== sentinel`
  tests, since real data is legitimately negative. Worth a lint/example fix.
- **Document `imagePullPolicy` semantics for plugin authors.** The fact that
  sideloaded images are honored (IfNotPresent) is the linchpin of the Thor
  workaround but is undocumented; authors should know how image resolution
  actually works on a node.
- **Camera-audio / network-sensor input patterns.** Auth quirks differ wildly
  (Reolink BCS/FLV needs query-param auth, NOT basic auth; Mobotix M16 MxPEG
  uses basic auth). A small "connecting to common cameras/sensors" cookbook
  would save every author the rediscovery.
- **Model-load vs. window-budget guidance.** Where bounded runtimes exist (§4),
  the wall-clock-vs-inference-time distinction (a ~28 GB model load eats minutes
  of the window) should be a documented first-class consideration.
