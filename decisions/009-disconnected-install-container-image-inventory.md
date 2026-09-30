# ADR: Disconnected Install Container Image Inventory (Operator and Helm)

## Context

**Problem**: RHDH customers on disconnected or restricted networks cannot reliably discover, mirror, and pull every **static container image** required for a supported install—including default-enabled capabilities such as Intelligent Assistant—because there is no single end-to-end contract from “what can run” to “what to mirror” to “what resolves at runtime.”

The RHDH Operator can cause pods to pull images defined not only in the ClusterServiceVersion operands but also in **flavour** and **plugin-deps** bundle ConfigMaps (for example `lightspeed-core`, OKP, Tekton UBI, Chainguard git). Today `spec.relatedImages` lists only a subset of those images, and disconnected tooling (`prepare-restricted-environment.sh`) does **not** treat `relatedImages` as the mirror source of truth—it discovers images mainly by processing CSV files inside operator bundles, without scanning flavour/plugin-deps manifests. Customer documentation still describes obsolete paths (manual bundle extract, legacy Lightspeed chart keys). Intelligent Assistant exposed this gap; it is a **class** problem, not a one-off image fix.

**Helm** installs use a different mechanism but the same customer need: every image referenced by chart defaults must be mirrorable and overridable. The `redhat-developer-hub` chart already encodes that contract in **`values.yaml`** (including `global.imageRegistry` and `intelligentAssistant.*`); product docs and mirror procedures are not consistently aligned with those values.

**Who is impacted**:

- Platform teams mirroring RHDH on OpenShift disconnected or partially disconnected clusters
- Teams using the supported prepare/mirror scripts and air-gap documentation
- Anyone deploying default or flavour-enabled Operator instances without hand-maintained `--extra-images` lists

**Constraints and scope**:

- **In scope**: Static container images the Operator or Helm chart can cause Kubernetes to pull (operands, sidecars, init containers, dependency workloads wired by bundle/chart defaults).
- **Out of scope (separate track)**: Dynamic plugin artifacts (`ref://`, OCI/npm) and `mirror-plugins.sh`—same customer journey, different artifact type.
- **Inclusion rule (Operator)**: If the Operator can cause a static container image to run as part of a supported deployment (including default-enabled flavours and optional add-ons when enabled), it belongs in the disconnected inventory.
- **Resolve (runtime)**: OpenShift may use registry mirroring (IDMS/ITMS) where the platform allows; image substitution in Operator reconcile is a follow-on only after standalone OpenShift validation that mirror config alone is sufficient when pod specs still reference upstream registry names.

**This repository**: `rhdh-adr` records the architectural decision only. Implementation, validation, and CI live in the repositories below. Claims about the **current** gap (incomplete `relatedImages`, tooling not consuming it, stale docs) were validated against those sources at the time this ADR was written; they are not reproduced as files in this repo.

**Relationship to [ADR 003](003-operator-plugin-config-processing.md)**: `ref://` plugin package resolution and OCI/npm plugin artifacts are **out of scope** for this static container inventory. Plugin mirroring uses separate tooling (`mirror-plugins.sh`) and documentation.

## Implementation repositories

| Concern | Owning repository | Canonical paths (main branch) | Planned validation / delivery |
|--------|-------------------|------------------------------|------------------------------|
| Operator **declare** (source manifests → bundle CSV) | [redhat-developer/rhdh-operator](https://github.com/redhat-developer/rhdh-operator) | Profile: `config/profile/rhdh/` (flavours, plugin-deps, default-config). Shipped CSV: `bundle/rhdh/manifests/backstage-operator.clusterserviceversion.yaml` (`spec.relatedImages`). Reconcile substitution today: `pkg/model/deployment.go`, `pkg/model/db-statefulset.go` | CI: generated bundle `relatedImages` matches profile static images (e.g. `hack/list-operator-container-images.sh` — **proposed**, not yet in tree) |
| Operator **discover / mirror** | [redhat-developer/rhdh-operator](https://github.com/redhat-developer/rhdh-operator) | `.rhdh/scripts/prepare-restricted-environment.sh`, `.rhdh/scripts/mirror-plugins.sh` (plugins only) | Tooling change: derive mirror set from CSV `spec.relatedImages` (+ manager image) |
| Operator **product docs** (in-repo) | [redhat-developer/rhdh-operator](https://github.com/redhat-developer/rhdh-operator) | `.rhdh/docs/airgap.adoc` | Align with inventory-first flow; propagate to customer docs |
| Helm **declare** | [redhat-developer/rhdh-chart](https://github.com/redhat-developer/rhdh-chart) | `charts/rhdh/values.yaml`, `charts/rhdh/README.md`, `charts/rhdh/docs/migration-from-backstage-chart.md` | Chart release tags bound inventory to a chart version; customers use `helm show values` for that version |
| Customer **mirror / install procedures** | [redhat-developer/red-hat-developers-documentation-rhdh](https://github.com/redhat-developer/red-hat-developers-documentation-rhdh) | Shared modules: `assemblies/modules/shared/proc-mirror-images-for-helm-deployments-on-*.adoc`, `proc-mirror-images-for-operator-deployments.adoc`; air-gap book: `assemblies/modules/install_installing-rhdh-in-an-air-gapped-environment/` | Doc updates only (no second inventory artifact for Helm) |

**Version boundary**: Operator disconnected inventory is tied to **operator bundle / OLM catalog version** (CSV in that bundle). Helm inventory is tied to **chart version** published with the product (e.g. `charts.openshift.io`). This ADR does not pin a specific release number; implementers apply it per release branch.

## Decision

Adopt a shared **Declare → Discover → Mirror → Resolve** model for disconnected **static container images**, with install-method-specific sources of truth:

| Stage | Operator (OLM) | Helm (`redhat-developer-hub` chart) |
|-------|----------------|--------------------------------------|
| **Declare** | Complete CSV `spec.relatedImages` (digest-pinned where practical), including operator manager image and all operator-caused static images (flavours, plugin-deps, operands) | Chart **`values.yaml` defaults** are the inventory; document how to derive the image list (`helm show values`) |
| **Discover** | Supported mirror tooling consumes **`spec.relatedImages`** (plus manager image if omitted), not ad hoc CSV grep or manual ConfigMap archaeology | Documentation and examples derive mirrors from chart defaults; no parallel “bundle extract” story |
| **Mirror** | Copy declared/discovered images to the customer registry via `prepare-restricted-environment.sh` / `oc-mirror` flows | Mirror images from values; use `global.imageRegistry` and per-key overrides (`intelligentAssistant.core.image`, etc.) |
| **Resolve** | Prefer platform mirror configuration where supported; defer expanding Operator env-based image substitution until validated on standalone OpenShift | Explicit values overrides and/or cluster registry configuration per platform docs |

**Implementation approach**:

**Operator**

- Maintain a **complete** `spec.relatedImages` set aligned with static images in `config/profile/rhdh` (and generated bundle), including default-enabled Intelligent Assistant and other flavour/plugin-deps images.
- Update **disconnected tooling** to build mirror and IDMS/ITMS inputs from `relatedImages` (and operator install image), with `--extra-images` reserved for site-specific exceptions—not undeclared product images.
- Add **CI guard**: generated bundle `relatedImages` must match a canonical list derived from operator profile manifests (fail on drift).
- Update **operator air-gap documentation** (`.rhdh/docs/airgap.adoc` and downstream product docs) to match Intelligent Assistant naming and the inventory-first flow.
- **Do not** broaden Operator reconcile image substitution until **Resolve** is tested on standalone OpenShift (mirror + IDMS/ITMS with unchanged upstream refs in pod specs).

**Helm**

- Treat **`helm show values`** on the shipped chart version as the customer-facing discovery step for static images.
- Align **product documentation** mirror procedures with `intelligentAssistant.*`, `global.imageRegistry`, and `ref://` plugin overrides (remove `global.lightspeed`, RAG init container, and obsolete OCI Lightspeed plugin examples).
- Cross-link **plugin mirroring** documentation; do not fold plugin OCI/npm into the static container inventory.

**Shared principles**

- One authoritative inventory per install method; tooling and docs must not contradict it.
- Default-enabled features must be mirrorable without tribal knowledge.
- Digest pinning in declarations where the release process allows, to stabilize disconnected mirroring.

## Alternatives Considered

### Alternative 1: Document-only fixes (`--extra-images`, manual bundle extract)
- **Approach**: Leave `relatedImages` and scripts unchanged; extend runbooks with per-image manual steps.
- **Rejected because**: Does not scale with flavours and plugin-deps; guarantees recurring gaps; contradicts OpenShift disconnected operator guidance.

### Alternative 2: Scan all bundle YAML for `image:` instead of `relatedImages`
- **Approach**: Teach the mirror script to walk every bundle manifest.
- **Rejected because**: Duplicates OLM’s intended contract, is fragile across packaging changes, and does not give customers a single CSV field to audit; may be used as a **temporary** safety net in tooling but not as the primary design.

### Alternative 3: Operator substitutes every image in reconciled manifests
- **Approach**: Immediately wire all sidecars and deps through `RELATED_IMAGE_*` env vars.
- **Rejected as first step because**: Resolve behavior on OpenShift with IDMS/ITMS is not yet validated; unnecessary complexity if platform mirroring suffices; decision gated on standalone cluster experiment.

### Alternative 4: Introduce a new Helm `relatedImages`-style CRD or file
- **Approach**: Duplicate Operator-style inventory for Helm.
- **Rejected because**: Chart values already serve that role; the gap is documentation and procedure alignment, not a missing artifact.

## Consequences

### Positive
- ✅ Disconnected installs have a **predictable**, auditable image list per install method
- ✅ Supported mirror scripts and OLM `relatedImages` align with OpenShift disconnected best practices
- ✅ Flavour and default-enabled features (e.g. Intelligent Assistant) are covered by the same pipeline as core operands
- ✅ Helm and Operator stories stay consistent at the principle level while respecting different mechanisms

### Negative
- ❌ Operator bundle and CSV maintenance burden increases (more `relatedImages` rows, digest pins, CI enforcement)
- ❌ Tooling refactor of `prepare-restricted-environment.sh` is required; until shipped, declaration alone does not fix mirroring
- ❌ Documentation churn across product docs and legacy Backstage chart references

### Neutral
- ⚖️ Dynamic plugins remain a parallel mirroring path (`mirror-plugins.sh` / plugin docs)
- ⚖️ Hosted control planes and clusters that block IDMS/ITMS may still need site-specific resolve strategies; this ADR does not mandate a single platform feature
- ⚖️ Implementation tracking lives in engineering backlogs; merging this ADR documents intent, not delivery dates
