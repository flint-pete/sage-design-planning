# sage-design-planning

Design docs, RFC drafts, and planning notes for the Sage/Waggle edge-computing work
— the "why" and "what next" behind the code that lives in the sibling repos
(`pywaggle2-nodeinfo`, `wes-nodeinfo-injection`, `wes-local-cache-manager`,
`image-sampler2`, `sage-yolo`, `sage-bioclip`, `birdnet`).

These are living planning documents, not deliverables. Implemented designs get
distilled into their own repo's `DESIGN.md`/`HANDOFF.md`; what remains here is the
roadmap for unbuilt work and the running issue/feature lists.

## Contents

| Doc | What it is |
|---|---|
| `pywaggle2-design.md` | The pywaggle2 library redesign (RFC draft). §2 node-identity has SHIPPED (see `pywaggle2-nodeinfo`); §1 acquisition ladder + §3 library enhancements are the still-unbuilt roadmap. |
| `Infra-problems-to-fix.md` | Authoritative running list of Sage **platform** bugs/gaps to raise upstream as GitHub issues. |
| `plugin-improvements.md` | Running list of enhancements to code **inside our plugins** (IS-1..IS-7 backlog). |
| `Sage-potential-features.md` | Earlier feature/interface-improvement brainstorm (largely sorted into the two lists above). |
| `sage-infra-issues-2026-06-18.md` | Point-in-time CI issues report from YOLO+BioCLIP dev on Thor H00F (folded into `Infra-problems-to-fix.md`). |
| `project-status.txt` | Documentation inventory / map of the observation/design/planning docs. |

## Cross-references

Docs reference each other by bare filename (they live together here). References from
the code repos point at `~/AI-projects/sage-design-planning/<doc>`. The big-picture
map of how all the pieces relate lives at `~/AI-projects/SAGE-STACK-MAP.md`.

The private `sage-hermes-brain` repo keeps its own frozen copies of some of these —
those are historical snapshots and are NOT updated from here.
