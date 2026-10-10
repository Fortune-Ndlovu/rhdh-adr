# ADR-009: Disconnected Install Container Image Inventory (Operator and Helm)

## Context

**Problem**: RHDH customers on disconnected or restricted networks cannot reliably discover, mirror, and pull every **static container image** required for a supported install—including default-enabled capabilities such as Intelligent Assistant—because there is no single end-to-end contract from "what can run" to "what to mirror" to "what resolves at runtime."

The RHDH Operator can cause pods to pull images defined not only in the ClusterServiceVersion operands but also in **flavour** and **plugin-deps** bundle ConfigMaps (for example `lightspeed-core`, OKP, Tekton UBI, Chainguard git). Today [`spec.relatedImages`](https://github.com/redhat-developer/rhdh-operator/blob/main/bundle/rhdh/manifests/backstage-operator.clusterserviceversion.yaml#L170-L177) lists only a subset of those images, and disconnected tooling ([`prepare-restricted-environment.sh`](https://github.com/redhat-developer/rhdh-operator/blob/main/.rhdh/scripts/prepare-restricted-environment.sh)) does **not** treat `relatedImages` as the mirror source of truth—it discovers images mainly by processing CSV files inside operator bundles, without scanning [flavour/plugin-deps manifests](https://github.com/redhat-developer/rhdh-operator/tree/main/config/profile/rhdh). Customer documentation still describes obsolete paths (manual bundle extract, legacy Lightspeed chart keys). Intelligent Assistant exposed this gap; it is a **class** problem, not a one-off image fix.

**Helm** installs use a different mechanism but the same customer need: every image referenced by chart defaults must be mirrorable and overridable. The `redhat-developer-hub` chart already encodes that contract in **[`values.yaml`](https://github.com/redhat-developer/rhdh-chart/blob/main/charts/redhat-developer-hub/values.yaml)** (including `global.imageRegistry` and `intelligentAssistant.*`); product docs and mirror procedures are not consistently aligned with those values.

### OpenShift Disconnected Operator Requirements

Per [OpenShift Container Platform documentation on OLM in disconnected environments](https://docs.openshift.com/container-platform/4.14/operators/admin/olm-restricted-networks.html), operators must meet the following criteria to run successfully in disconnected environments:

> "List any related images, or other container images that the Operator might require to perform their functions, in the **relatedImages** parameter of its ClusterServiceVersion (CSV) object. Reference all specified images by a **digest (SHA)** and not by a tag."

RHDH currently does not fully comply with this pattern for flavor and plugin-dependency images.

### Evidence of Current Gap

From current [RHDH 1.10 LightSpeed air-gap documentation](https://docs.redhat.com/en/documentation/red_hat_developer_hub/1.10/html/interacting_with_red_hat_developer_lightspeed_for_red_hat_developer_hub/configure-to-initialize-the-ai-assistant_interacting-with-developer-lightspeed-for-rhdh#mirror-images-for-air-gapped-environments_configure-to-initialize-the-ai-assistant):

> "The `prepare-restricted-environment.sh` script does **not automatically parse Developer Lightspeed for RHDH images from the bundle manifest**, so mirror these images manually before running the script."

This manual extraction requirement demonstrates the architectural gap this ADR addresses.

**Who is impacted**:

- Platform teams mirroring RHDH on OpenShift disconnected or partially disconnected clusters
- Teams using the supported prepare/mirror scripts and air-gap documentation
- Anyone deploying default or flavour-enabled Operator instances without hand-maintained `--extra-images` lists

**Constraints and scope**:

- **In scope**: Static container images the Operator or Helm chart can cause Kubernetes to pull (operands, sidecars, init containers, dependency workloads wired by bundle/chart defaults).
- **Out of scope (separate track)**: Dynamic plugin artifacts (`ref://`, OCI/npm) and `mirror-plugins.sh`—same customer journey, different artifact type.
- **Inclusion rule (Operator)**: If the Operator can cause a static container image to run as part of a supported deployment (including default-enabled flavours and optional add-ons when enabled), it belongs in the disconnected inventory.
- **Orchestrator images**: Include orchestrator container images in `relatedImages`; OpenShift Serverless operators are documented as **prerequisites** users install separately (see [Orchestrator air-gap documentation](https://docs.redhat.com/en/documentation/red_hat_developer_hub/1.10/html/orchestrator_in_red_hat_developer_hub/install-orchestrator-plugin-in-an-air-gapped-environment-with-the-operator_orchestrator-in-rhdh)).
- **Resolve (runtime)**: OpenShift may use registry mirroring (IDMS/ITMS) where the platform allows; image substitution in Operator reconcile is a follow-on only after standalone OpenShift validation that mirror config alone is sufficient when pod specs still reference upstream registry names.

**This repository**: `rhdh-adr` records the architectural decision only. Implementation, validation, and CI live in the repositories below. Claims about the **current** gap (incomplete `relatedImages`, tooling not consuming it, stale docs) were validated against those sources at the time this ADR was written; they are not reproduced as files in this repo.

**Relationship to [ADR 003](003-operator-plugin-config-processing.md)**: `ref://` plugin package resolution and OCI/npm plugin artifacts are **out of scope** for this static container inventory. Plugin mirroring uses separate tooling (`mirror-plugins.sh`) and documentation.

## Implementation repositories

| Concern | Owning repository | Canonical paths (main branch) | Planned validation / delivery |
|--------|-------------------|------------------------------|------------------------------|
| Operator **declare** (source manifests → bundle CSV) | [redhat-developer/rhdh-operator](https://github.com/redhat-developer/rhdh-operator) | Profile: [`config/profile/rhdh/`](https://github.com/redhat-developer/rhdh-operator/tree/main/config/profile/rhdh) (flavours, plugin-deps, default-config). Shipped CSV: [`bundle/rhdh/manifests/backstage-operator.clusterserviceversion.yaml`](https://github.com/redhat-developer/rhdh-operator/blob/main/bundle/rhdh/manifests/backstage-operator.clusterserviceversion.yaml) ([`spec.relatedImages`](https://github.com/redhat-developer/rhdh-operator/blob/main/bundle/rhdh/manifests/backstage-operator.clusterserviceversion.yaml#L170-L177) + operator deployment [`RELATED_IMAGE_*` env vars](https://github.com/redhat-developer/rhdh-operator/blob/main/bundle/rhdh/manifests/backstage-operator.clusterserviceversion.yaml#L311-L316)). Reconcile substitution: [`pkg/model/deployment.go`](https://github.com/redhat-developer/rhdh-operator/blob/main/pkg/model/deployment.go#L112-L114), [`pkg/model/db-statefulset.go`](https://github.com/redhat-developer/rhdh-operator/blob/main/pkg/model/db-statefulset.go#L17) | Existing pattern extends to flavor images |
| Operator **discover / mirror** | [redhat-developer/rhdh-operator](https://github.com/redhat-developer/rhdh-operator) | [`.rhdh/scripts/prepare-restricted-environment.sh`](https://github.com/redhat-developer/rhdh-operator/blob/main/.rhdh/scripts/prepare-restricted-environment.sh), [`.rhdh/scripts/mirror-plugins.sh`](https://github.com/redhat-developer/rhdh-operator/blob/main/.rhdh/scripts/mirror-plugins.sh) (plugins only) | Tooling change: prefer CSV `spec.relatedImages`, fallback to old parsing for backward compatibility |
| Operator **product docs** (in-repo) | [redhat-developer/rhdh-operator](https://github.com/redhat-developer/rhdh-operator) | `.rhdh/docs/airgap.adoc` | Align with inventory-first flow; propagate to customer docs |
| Helm **declare** | [redhat-developer/rhdh-chart](https://github.com/redhat-developer/rhdh-chart) | [`charts/redhat-developer-hub/values.yaml`](https://github.com/redhat-developer/rhdh-chart/blob/main/charts/redhat-developer-hub/values.yaml), [`README.md`](https://github.com/redhat-developer/rhdh-chart/blob/main/charts/redhat-developer-hub/README.md) | Chart release tags bind inventory to a chart version; customers use `helm show values` for that version |
| Customer **mirror / install procedures** | [redhat-developer/red-hat-developers-documentation-rhdh](https://github.com/redhat-developer/red-hat-developers-documentation-rhdh) | Shared modules: `assemblies/modules/shared/proc-mirror-images-for-helm-deployments-on-*.adoc`, `proc-mirror-images-for-operator-deployments.adoc`; air-gap book: `assemblies/modules/install_installing-rhdh-in-an-air-gapped-environment/` | Doc updates only (no second inventory artifact for Helm) |

**Version boundary**: Operator disconnected inventory is tied to **operator bundle / OLM catalog version** (CSV in that bundle). Helm inventory is tied to **chart version** published with the product (e.g. `charts.openshift.io`). This ADR does not pin a specific release number; implementers apply it per release branch.

## Complete Image Inventory

This section provides the comprehensive list of container images that will be included in `spec.relatedImages`, clarifying what RHDH provides vs. what are external prerequisites.

### Current Images (Already in relatedImages)

| Image Name | Registry Reference (Upstream Example) | Purpose | CSV Entry |
|------------|--------------------------------------|---------|-----------|
| `postgresql` | `quay.io/fedora/postgresql-18:latest` | Local database (when `enableLocalDb: true`) | [View in CSV](https://github.com/redhat-developer/rhdh-operator/blob/main/bundle/rhdh/manifests/backstage-operator.clusterserviceversion.yaml#L171-L172) |
| `backstage` | `quay.io/rhdh-community/rhdh:next` | Main RHDH backend container | [View in CSV](https://github.com/redhat-developer/rhdh-operator/blob/main/bundle/rhdh/manifests/backstage-operator.clusterserviceversion.yaml#L173-L174) |
| `plugin-installer` | `quay.io/rhdh-community/rhdh-plugin-installer:next` | Dynamic plugin installation init container | [View in CSV](https://github.com/redhat-developer/rhdh-operator/blob/main/bundle/rhdh/manifests/backstage-operator.clusterserviceversion.yaml#L175-L176) |

**Current Status**: ✅ These images are properly declared and handled via `RELATED_IMAGE_*` environment variables.

### New Images Being Added (LightSpeed / Intelligent Assistant)

**Source**: [LightSpeed flavor ConfigMap](https://github.com/redhat-developer/rhdh-operator/tree/main/config/profile/rhdh/flavour-lightspeed)

| Image Name | Registry Reference (Downstream Example) | Purpose | Will Add To |
|------------|----------------------------------------|---------|-------------|
| `lightspeed-rag-init` | `registry.redhat.io/rhdh/rhdh-lightspeed-rag-content-init-rhel9@sha256:...` | LightSpeed RAG data initialization (init container) | `spec.relatedImages` + `RELATED_IMAGE_lightspeed_rag_init` env var |
| `lightspeed-core` | `registry.redhat.io/rhdh/rhdh-lightspeed-service-rhel9@sha256:...` | LightSpeed backend service (sidecar container) | `spec.relatedImages` + `RELATED_IMAGE_lightspeed_core` env var |

**Helm Equivalent**: [`intelligentAssistant.initContainer.image`](https://github.com/redhat-developer/rhdh-chart/blob/main/charts/redhat-developer-hub/values.yaml#L218-L222) and [`intelligentAssistant.sidecar.image`](https://github.com/redhat-developer/rhdh-chart/blob/main/charts/redhat-developer-hub/values.yaml#L223-L227)

**User Action**: Mirror these images alongside core RHDH images. No separate prerequisite installation required.

**Current Gap**: [LightSpeed air-gap documentation](https://docs.redhat.com/en/documentation/red_hat_developer_hub/1.10/html/interacting_with_red_hat_developer_lightspeed_for_red_hat_developer_hub/configure-to-initialize-the-ai-assistant_interacting-with-developer-lightspeed-for-rhdh#mirror-images-for-air-gapped-environments_configure-to-initialize-the-ai-assistant) currently requires manual extraction from bundle ConfigMaps.

### Orchestrator Infrastructure Images

**Important Distinction**: 
- **Container Images** (in scope for this ADR) - static containers RHDH operator deploys
- **Plugin OCI Artifacts** (out of scope) - handled by [`mirror-plugins.sh`](https://github.com/redhat-developer/rhdh-operator/blob/main/.rhdh/scripts/mirror-plugins.sh)
- **Prerequisite Operators** (out of scope) - external dependencies documented separately

**Note**: The orchestrator feature primarily uses **dynamic plugin OCI artifacts**, not static container images. Investigation needed to determine if any static container images are required beyond the plugin artifacts.

**Plugin OCI Artifacts** (handled separately via `mirror-plugins.sh`):
```yaml
# These are plugin packages, NOT container images for pods
# Mirrored via mirror-plugins.sh, not in spec.relatedImages
- registry.access.redhat.com/rhdh/red-hat-developer-hub-backstage-plugin-orchestrator@sha256:...
- registry.access.redhat.com/rhdh/red-hat-developer-hub-backstage-plugin-orchestrator-backend@sha256:...
- registry.access.redhat.com/rhdh/red-hat-developer-hub-backstage-plugin-scaffolder-backend-module-orchestrator@sha256:...
- registry.access.redhat.com/rhdh/red-hat-developer-hub-backstage-plugin-orchestrator-form-widgets@sha256:...
```

**Reference**: See [Orchestrator air-gap documentation](https://docs.redhat.com/en/documentation/red_hat_developer_hub/1.10/html/orchestrator_in_red_hat_developer_hub/install-orchestrator-plugin-in-an-air-gapped-environment-with-the-operator_orchestrator-in-rhdh#L28-L33) `ImageSetConfiguration` `additionalImages` section.

### Orchestrator Prerequisites (External - Not in RHDH relatedImages)

These operators are **external dependencies** that users must install and mirror separately **before** using the Orchestrator feature:

| Operator | Package Name | Purpose | Mirror Instructions |
|----------|-------------|---------|---------------------|
| OpenShift Serverless | `serverless-operator` | Knative Eventing and Serving | [OpenShift Serverless Documentation](https://docs.openshift.com/serverless/) |
| OpenShift Serverless Logic | `logic-operator` | Serverless workflow execution | [Included in orchestrator prerequisites](https://docs.redhat.com/en/documentation/red_hat_developer_hub/1.10/html/orchestrator_in_red_hat_developer_hub/install-orchestrator-plugin-in-an-air-gapped-environment-with-the-operator_orchestrator-in-rhdh#L36-L48) |

**User Action**: Users mirror these operators using their own `ImageSetConfiguration`:

```yaml
# Example from orchestrator air-gap documentation
apiVersion: mirror.openshift.io/v2alpha1
kind: ImageSetConfiguration
mirror:
  operators:
    - catalog: registry.redhat.io/redhat/redhat-operator-index:v4.21
      packages:
      - name: logic-operator
        channels:
        - name: stable
          minVersion: 1.38.1
          maxVersion: 1.38.1
      - name: serverless-operator
        channels:
        - name: stable
          minVersion: 1.37.1
          maxVersion: 1.37.1
```

**Reference**: [Orchestrator air-gap documentation - serverless prerequisites](https://docs.redhat.com/en/documentation/red_hat_developer_hub/1.10/html/orchestrator_in_red_hat_developer_hub/install-orchestrator-plugin-in-an-air-gapped-environment-with-the-operator_orchestrator-in-rhdh)

**Why Not in RHDH relatedImages**: 
- These are independent operators with their own OLM lifecycle
- They have their own `relatedImages` in their respective CSVs
- They must be installed cluster-wide before RHDH orchestrator can use them
- Their images are managed by the OpenShift Serverless team, not RHDH

### Additional Flavor/Plugin-Dependency Images (Investigation Required)

**Source**: [`config/profile/rhdh/plugin-deps/`](https://github.com/redhat-developer/rhdh-operator/tree/main/config/profile/rhdh/plugin-deps)

Potential additional images from plugin dependencies (examples):
- Tekton pipeline helper images
- OKP (OpenShift Kubernetes Plugin) images
- Other Git/SCM helper containers

**Action Item**: Audit [`config/profile/rhdh/`](https://github.com/redhat-developer/rhdh-operator/tree/main/config/profile/rhdh) manifests to identify all static container images referenced in:
- `flavour-*/` directories
- `plugin-deps/` ConfigMaps
- `default-config/` deployment patches

### Summary: What Goes Where

```
┌─────────────────────────────────────────────────────────────────┐
│ RHDH Operator spec.relatedImages (THIS ADR)                    │
├─────────────────────────────────────────────────────────────────┤
│ ✅ PostgreSQL                     (already included)            │
│ ✅ RHDH Backend                   (already included)            │
│ ✅ Plugin Installer                (already included)            │
│ ➕ LightSpeed RAG Init            (adding)                      │
│ ➕ LightSpeed Core Service        (adding)                      │
│ ➕ Other flavor/plugin-dep images (to be identified)            │
└─────────────────────────────────────────────────────────────────┘
                            ↓
            User mirrors via prepare-restricted-environment.sh
                            ↓
            Images available in customer registry

┌─────────────────────────────────────────────────────────────────┐
│ Dynamic Plugin OCI Artifacts (Separate: mirror-plugins.sh)      │
├─────────────────────────────────────────────────────────────────┤
│ ⚙️ Orchestrator plugins (4 packages)                            │
│ ⚙️ Other dynamic plugins from catalog index                     │
└─────────────────────────────────────────────────────────────────┘
                            ↓
            User mirrors via mirror-plugins.sh
                            ↓
            Plugins available in customer registry

┌─────────────────────────────────────────────────────────────────┐
│ External Operator Prerequisites (Separate: OpenShift Docs)      │
├─────────────────────────────────────────────────────────────────┤
│ 🔧 OpenShift Serverless Operator                                │
│ 🔧 OpenShift Serverless Logic Operator                          │
└─────────────────────────────────────────────────────────────────┘
                            ↓
            User mirrors via oc-mirror (operator catalogs)
                            ↓
            Operators installed via OperatorHub
```

**Documentation Cross-Reference**:
- Container images → This ADR + [operator air-gap docs](https://github.com/redhat-developer/rhdh-operator/blob/main/.rhdh/docs/airgap.adoc)
- Plugin artifacts → [mirror-plugins.sh documentation](https://github.com/redhat-developer/rhdh-operator/blob/main/.rhdh/scripts/mirror-plugins.sh)
- Serverless prerequisites → [OpenShift Serverless air-gap documentation](https://docs.openshift.com/serverless/)

## Decision

Adopt a shared **Declare → Discover → Mirror → Resolve** model for disconnected **static container images**, with install-method-specific sources of truth:

| Stage | Operator (OLM) | Helm (`redhat-developer-hub` chart) |
|-------|----------------|--------------------------------------|
| **Declare** | Complete CSV `spec.relatedImages` (digest-pinned where practical) + `RELATED_IMAGE_*` environment variables in operator deployment, including all operator-caused static images (flavours, plugin-deps, operands, orchestrator). Extends existing pattern. | Chart **`values.yaml` defaults** are the inventory; document how to derive the image list (`helm show values`) |
| **Discover** | Supported mirror tooling reads **`spec.relatedImages`** first, falls back to ad-hoc CSV parsing for backward compatibility with older bundles | Documentation and examples derive mirrors from chart defaults; no parallel "bundle extract" story |
| **Mirror** | Copy declared/discovered images to the customer registry via `prepare-restricted-environment.sh` / `oc-mirror` flows | Mirror images from values; use `global.imageRegistry` and per-key overrides (`intelligentAssistant.core.image`, etc.) |
| **Resolve** | Prefer platform mirror configuration (IDMS/ITMS) where supported; users can override via Subscription `spec.config.env` for RELATED_IMAGE vars (OLM v0) or equivalent ClusterExtension configuration (OLM v1) | Explicit values overrides and/or cluster registry configuration per platform docs |

### Operator RELATED_IMAGE Environment Variable Pattern

The operator **already implements** the OpenShift disconnected operator pattern for core images (PostgreSQL, Backstage). This ADR **extends** that pattern to flavor and optional component images.

#### How It Works

1. **CSV Declaration** - Both [`spec.relatedImages`](https://github.com/redhat-developer/rhdh-operator/blob/main/bundle/rhdh/manifests/backstage-operator.clusterserviceversion.yaml#L170-L177) and [operator deployment env vars](https://github.com/redhat-developer/rhdh-operator/blob/main/bundle/rhdh/manifests/backstage-operator.clusterserviceversion.yaml#L311-L316):

```yaml
# bundle/rhdh/manifests/backstage-operator.clusterserviceversion.yaml
apiVersion: operators.coreos.com/v1alpha1
kind: ClusterServiceVersion
spec:
  relatedImages:
  - image: registry.redhat.io/rhdh/rhdh-hub-rhel9@sha256:abc123...
    name: backstage
  - image: registry.redhat.io/rhdh/rhdh-rhel9-postgresql-15@sha256:def456...
    name: postgresql
  - image: registry.redhat.io/rhdh/lightspeed-core-rhel9@sha256:ghi789...
    name: lightspeed-core
  
  install:
    spec:
      deployments:
      - spec:
          template:
            spec:
              containers:
              - name: manager
                env:
                - name: RELATED_IMAGE_backstage
                  value: registry.redhat.io/rhdh/rhdh-hub-rhel9@sha256:abc123...
                - name: RELATED_IMAGE_postgresql
                  value: registry.redhat.io/rhdh/rhdh-rhel9-postgresql-15@sha256:def456...
                - name: RELATED_IMAGE_lightspeed_core
                  value: registry.redhat.io/rhdh/lightspeed-core-rhel9@sha256:ghi789...
```

2. **Operator Code** - Reads environment variables at runtime (existing pattern in [`pkg/model/deployment.go`](https://github.com/redhat-developer/rhdh-operator/blob/main/pkg/model/deployment.go#L31) and [`pkg/model/db-statefulset.go`](https://github.com/redhat-developer/rhdh-operator/blob/main/pkg/model/db-statefulset.go#L17)):

```go
// pkg/model/deployment.go
const BackstageImageEnvVar = "RELATED_IMAGE_backstage"

// At reconciliation time:
if os.Getenv(BackstageImageEnvVar) != "" {
    b.setImage(ptr.To(os.Getenv(BackstageImageEnvVar)), i)
}
```

3. **User Override** - Via [Subscription](https://github.com/redhat-developer/rhdh-operator/blob/main/config/samples/catalog-subscription-template.yaml) (OLM v0) or [ClusterExtension](https://github.com/redhat-developer/rhdh-operator/blob/main/config/samples/catalog-cluster-extension-template.yaml) (OLM v1):

**OLM v0 - Subscription with Environment Variable Override:**

```yaml
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: rhdh-operator
  namespace: rhdh-operator
spec:
  channel: stable
  name: rhdh-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
  config:
    env:
      - name: RELATED_IMAGE_backstage
        value: mirror.example.com/rhdh/rhdh-hub-rhel9@sha256:abc123...
      - name: RELATED_IMAGE_postgresql
        value: mirror.example.com/rhdh/postgresql-15@sha256:def456...
      - name: RELATED_IMAGE_lightspeed_core
        value: mirror.example.com/rhdh/lightspeed-core-rhel9@sha256:ghi789...
      - name: RELATED_IMAGE_lightspeed_rag_init
        value: mirror.example.com/rhdh/lightspeed-rag-init@sha256:jkl012...
```

**OLM v1 - ClusterExtension with Environment Variable Override:**

In OLM v1, environment variable overrides are applied differently. The recommended approach is to patch the operator deployment after ClusterExtension creates it, or configure via the operator's configuration mechanism.

```yaml
# ClusterExtension creates the operator deployment
apiVersion: olm.operatorframework.io/v1
kind: ClusterExtension
metadata:
  name: rhdh-operator
spec:
  namespace: rhdh-operator
  serviceAccount:
    name: rhdh-operator-installer
  source:
    sourceType: Catalog
    catalog:
      packageName: rhdh-operator
      selector:
        matchLabels:
          olm.operatorframework.io/metadata.name: rhdh-operator
  install:
    preflight:
      crdUpgradeSafety:
        enforcement: None
```

After the ClusterExtension deploys the operator, users can patch the deployment:

```bash
# Patch operator deployment to override images
kubectl patch deployment rhdh-operator-controller-manager \
  -n rhdh-operator \
  --type='json' \
  -p='[
    {
      "op": "add",
      "path": "/spec/template/spec/containers/0/env/-",
      "value": {
        "name": "RELATED_IMAGE_backstage",
        "value": "mirror.example.com/rhdh/rhdh-hub-rhel9@sha256:abc123..."
      }
    },
    {
      "op": "add",
      "path": "/spec/template/spec/containers/0/env/-",
      "value": {
        "name": "RELATED_IMAGE_lightspeed_core",
        "value": "mirror.example.com/rhdh/lightspeed-core-rhel9@sha256:ghi789..."
      }
    }
  ]'
```

**Note**: OLM v1 architecture is evolving. The preferred method for environment variable overrides in ClusterExtension may change. For production deployments:
- **Prefer IDMS/ITMS** (ImageDigestMirrorSet/ImageTagMirrorSet) for transparent registry mirroring
- Use deployment patches only when IDMS/ITMS is not available (e.g., hosted control planes)
- Consult [OpenShift OLM v1 documentation](https://docs.openshift.com/container-platform/4.21/operators/understanding/olm/olm-packaging-format.html) for latest best practices

See [rhdh-operator admin documentation](https://github.com/redhat-developer/rhdh-operator/blob/main/docs/admin.md#airgapped-environment) for complete examples.

**Implementation approach**:

**Operator**

- Extend the existing [`RELATED_IMAGE_*` pattern](https://github.com/redhat-developer/rhdh-operator/blob/main/bundle/rhdh/manifests/backstage-operator.clusterserviceversion.yaml#L311-L316) to include:
  - LightSpeed images (init container, sidecar from [flavour ConfigMap](https://github.com/redhat-developer/rhdh-operator/tree/main/config/profile/rhdh/flavour-lightspeed))
  - Orchestrator images (excluding serverless operator prerequisites)
  - Additional flavor and plugin-dependency images from [`config/profile/rhdh/`](https://github.com/redhat-developer/rhdh-operator/tree/main/config/profile/rhdh)

- Update **operator air-gap documentation** (`.rhdh/docs/airgap.adoc` and downstream product docs) with **manual-first approach**:
  - **Primary documentation**: Step-by-step manual commands using standard tooling:
    - Extract image list from CSV `spec.relatedImages` using `yq` or `jq`
    - Mirror images using standard tools (`oc-mirror` v2 on OpenShift, `skopeo` on Kubernetes)
    - Apply IDMS/ITMS or configure Subscription overrides
    - Clear, transparent bash snippets (10-20 lines max) that users can understand and customize
  - **Secondary option**: Link to [`prepare-restricted-environment.sh`](https://github.com/redhat-developer/rhdh-operator/blob/main/.rhdh/scripts/prepare-restricted-environment.sh) as optional automation for those who prefer it
  - **Rationale**: Enterprise customers often have security policies against running large external scripts; they prefer transparent, auditable commands they control
  - Document the Subscription `spec.config.env` override pattern with examples
  - Reference OLM v1 ClusterExtension equivalent configuration
  - Align LightSpeed documentation with `relatedImages` approach (remove manual extraction steps)

- Update **disconnected tooling** ([`prepare-restricted-environment.sh`](https://github.com/redhat-developer/rhdh-operator/blob/main/.rhdh/scripts/prepare-restricted-environment.sh)) to:
  - Read from [`spec.relatedImages`](https://github.com/redhat-developer/rhdh-operator/blob/main/bundle/rhdh/manifests/backstage-operator.clusterserviceversion.yaml#L170-L177) as primary source
  - Maintain fallback to old CSV parsing for backward compatibility with previous install plans
  - Keep `--extra-images` for site-specific exceptions—not undeclared product images
  - **Position in docs**: Optional automation layer, not the primary procedure

- **Note**: Operator reconcile already handles environment variable substitution for core images; extending to additional images uses the same code paths.

**Helm**

- Treat **`helm show values`** on the shipped chart version as the customer-facing discovery step for static images.

- Align **product documentation** mirror procedures with current [chart structure](https://github.com/redhat-developer/rhdh-chart/blob/main/charts/redhat-developer-hub/values.yaml) using **manual-first approach**:
  - **Primary documentation**: Clear manual steps:
    - Extract image references from chart values using `helm show values` and `yq`
    - Mirror images using standard tools (`oc-mirror` v2 on OpenShift, `skopeo` on Kubernetes)
    - Override images in custom `values.yaml` using standard Helm mechanisms
    - Provide concise, copy-paste-ready command examples
  - **No large scripts required**: Helm chart values already serve as the image inventory; extraction is straightforward
  - Document image override keys:
    - [`intelligentAssistant.initContainer.image.*`](https://github.com/redhat-developer/rhdh-chart/blob/main/charts/redhat-developer-hub/values.yaml#L218-L222)
    - [`intelligentAssistant.sidecar.image.*`](https://github.com/redhat-developer/rhdh-chart/blob/main/charts/redhat-developer-hub/values.yaml#L223-L227)
    - [`global.imageRegistry`](https://github.com/redhat-developer/rhdh-chart/blob/main/charts/redhat-developer-hub/values.yaml#L10) for registry-wide overrides
    - `ref://` plugin overrides (separate from static container images)

- Remove obsolete references:
  - `global.lightspeed.*` (deprecated keys)
  - RAG init container examples not aligned with current chart
  - OCI Lightspeed plugin examples that don't match current defaults

- Cross-link **plugin mirroring** documentation with same manual-first approach:
  - Show manual `skopeo` commands for plugin OCI artifacts
  - Link to [`mirror-plugins.sh`](https://github.com/redhat-developer/rhdh-operator/blob/main/.rhdh/scripts/mirror-plugins.sh) as optional automation
  - Do not fold plugin OCI/npm into the static container inventory

**Shared principles**

- One authoritative inventory per install method; tooling and docs must not contradict it.
- Default-enabled features must be mirrorable without tribal knowledge.
- Digest pinning in declarations where the release process allows (downstream), tags where necessary (upstream).
- Backward compatibility in tooling maintains support for prior operator versions during transition.

## Alternatives Considered

### Alternative 1: Document-only fixes (`--extra-images`, manual bundle extract)
- **Approach**: Leave `relatedImages` and scripts unchanged; extend runbooks with per-image manual steps.
- **Rejected because**: Does not scale with flavours and plugin-deps; guarantees recurring gaps; contradicts OpenShift disconnected operator guidance.

### Alternative 2: Scan all bundle YAML for `image:` instead of `relatedImages`
- **Approach**: Teach the mirror script to walk every bundle manifest.
- **Rejected because**: Duplicates OLM's intended contract, is fragile across packaging changes, and does not give customers a single CSV field to audit; may be used as a **temporary** safety net in tooling but not as the primary design.

### Alternative 3: Operator substitutes every image in reconciled manifests
- **Approach**: Immediately wire all sidecars and deps through `RELATED_IMAGE_*` env vars.
- **Rejected as first step because**: Operator already implements this pattern for core images; this ADR extends it to additional images. No architectural change needed, only scope expansion.

### Alternative 4: Introduce a new Helm `relatedImages`-style CRD or file
- **Approach**: Duplicate Operator-style inventory for Helm.
- **Rejected because**: Chart values already serve that role; the gap is documentation and procedure alignment, not a missing artifact.

## Consequences

### Positive
- ✅ Disconnected installs have a **predictable**, auditable image list per install method
- ✅ Supported mirror scripts and OLM `relatedImages` align with OpenShift disconnected best practices
- ✅ Flavour and default-enabled features (e.g. Intelligent Assistant) are covered by the same pipeline as core operands
- ✅ Helm and Operator stories stay consistent at the principle level while respecting different mechanisms
- ✅ Backward-compatible tooling approach allows gradual transition without breaking older install plans
- ✅ Subscription/ClusterExtension override mechanism provides escape hatch for users with hosted control planes or other IDMS/ITMS restrictions

### Negative
- ❌ Operator bundle and CSV maintenance burden increases (more `relatedImages` rows, more env vars, digest pins)
- ❌ Tooling refactor of `prepare-restricted-environment.sh` required to prefer `relatedImages`; until shipped, declaration alone does not fix mirroring
- ❌ Documentation churn across product docs and legacy chart references

### Neutral
- ⚖️ Dynamic plugins remain a parallel mirroring path (`mirror-plugins.sh` / plugin docs)
- ⚖️ Hosted control planes and clusters that block IDMS/ITMS can use Subscription `spec.config.env` overrides; this ADR does not mandate a single platform feature
- ⚖️ Implementation tracking lives in engineering backlogs; merging this ADR documents intent, not delivery dates
- ⚖️ Backward compatibility fallback in tooling maintains support during transition period

## Documentation Update Plan

### Operator Air-Gap Documentation

**File**: [`rhdh-operator/.rhdh/docs/airgap.adoc`](https://github.com/redhat-developer/rhdh-operator/blob/main/.rhdh/docs/airgap.adoc)

**Updates Needed**:
1. Add section explaining `spec.relatedImages` as source of truth
2. Document Subscription `spec.config.env` override pattern with examples (see [`docs/admin.md`](https://github.com/redhat-developer/rhdh-operator/blob/main/docs/admin.md#airgapped-environment))
3. Add OLM v1 ClusterExtension configuration notes
4. Update LightSpeed section to reference `relatedImages` (remove manual extraction steps)
5. Add orchestrator image handling with link to [OpenShift Serverless documentation](https://docs.openshift.com/serverless/)

**Cross-links**:
- Link to [OpenShift OLM disconnected environments documentation](https://docs.openshift.com/container-platform/4.14/operators/admin/olm-restricted-networks.html)
- Link to [Operator SDK CSV generation documentation](https://sdk.operatorframework.io/docs/olm-integration/generation/#clusterserviceversion-manifests)
- Link to customer documentation (once updated)

### Customer Product Documentation

**Repository**: [`red-hat-developers-documentation-rhdh`](https://github.com/redhat-developer/red-hat-developers-documentation-rhdh)

**Base Documentation**: [Installing RHDH in an Air-Gapped Environment](https://docs.redhat.com/en/documentation/red_hat_developer_hub/1.10/html-single/installing_red_hat_developer_hub_in_an_air-gapped_environment/)

**Updates Needed**:

1. **Operator Installation Procedures** ([current docs](https://docs.redhat.com/en/documentation/red_hat_developer_hub/1.10/html-single/installing_red_hat_developer_hub_in_an_air-gapped_environment/index#install-rhdh-in-an-air-gapped-environment-with-the-operator_installing-rhdh-in-an-air-gapped-environment)):
   
   **Documentation Structure (Manual-First Approach)**:
   
   - **Primary Path - Manual Steps**:
     ```bash
     # Step 1: Extract image list from CSV
     CSV_URL="https://raw.githubusercontent.com/redhat-developer/rhdh-operator/refs/tags/v2.1.0/bundle/rhdh/manifests/backstage-operator.clusterserviceversion.yaml"
     curl -sSL $CSV_URL | yq '.spec.relatedImages[].image'
     
     # Step 2: Mirror images (OpenShift example)
     cat > imageset.yaml <<EOF
     apiVersion: mirror.openshift.io/v2alpha1
     kind: ImageSetConfiguration
     mirror:
       additionalImages:
       - name: registry.redhat.io/rhdh/rhdh-hub-rhel9@sha256:abc...
       - name: registry.redhat.io/rhdh/rhdh-lightspeed-service-rhel9@sha256:def...
     EOF
     oc-mirror --config imageset.yaml docker://mirror.example.com
     
     # Step 3: Apply IDMS (generated by oc-mirror)
     oc apply -f oc-mirror-workspace/results-*/imageContentSourcePolicy.yaml
     ```
   
   - **Alternative Path - Kubernetes/Skopeo**:
     ```bash
     # For each image from relatedImages:
     skopeo copy \
       docker://registry.redhat.io/rhdh/rhdh-hub-rhel9@sha256:abc... \
       docker://mirror.example.com/rhdh/rhdh-hub-rhel9@sha256:abc...
     ```
   
   - **Optional Automation**:
     > For users who prefer automation, the operator repository provides [`prepare-restricted-environment.sh`](https://github.com/redhat-developer/rhdh-operator/blob/main/.rhdh/scripts/prepare-restricted-environment.sh) which automates the steps above. This script requires bash, yq, skopeo, and other dependencies. Review the script source before running in production environments.
   
   - Document Subscription `spec.config.env` override pattern
   - Reference OLM v1 ClusterExtension deployment patch approach
   - Cross-link to [operator repository documentation](https://github.com/redhat-developer/rhdh-operator/blob/main/.rhdh/docs/airgap.adoc)

2. **Helm Installation Procedures** ([current docs](https://docs.redhat.com/en/documentation/red_hat_developer_hub/1.10/html-single/installing_red_hat_developer_hub_in_an_air-gapped_environment/index#install-rhdh-on-ocp-in-an-air-gapped-environment-with-the-helm-chart_installing-rhdh-in-an-air-gapped-environment)):
   
   **Documentation Structure (Manual-First Approach)**:
   
   - **Primary Path - Manual Steps**:
     ```bash
     # Step 1: Extract image references from chart
     helm repo add openshift-charts https://charts.openshift.io
     helm show values openshift-charts/redhat-developer-hub --version 2.1.0 > values.yaml
     
     # Extract main images
     RHDH_IMAGE=$(yq '.upstream.backstage.image.registry + "/" + .upstream.backstage.image.repository + "@" + .upstream.backstage.image.tag' values.yaml)
     PG_IMAGE=$(yq '.upstream.postgresql.image.registry + "/" + .upstream.postgresql.image.repository + "@" + .upstream.postgresql.image.tag' values.yaml)
     LS_CORE=$(yq '.intelligentAssistant.sidecar.image.registry + "/" + .intelligentAssistant.sidecar.image.repository + "@" + .intelligentAssistant.sidecar.image.tag' values.yaml)
     
     # Step 2: Mirror images
     for img in "$RHDH_IMAGE" "$PG_IMAGE" "$LS_CORE"; do
       skopeo copy --all docker://$img docker://mirror.example.com/${img#*/}
     done
     
     # Step 3: Create custom values.yaml with mirror registry
     cat > values-airgap.yaml <<EOF
     global:
       imageRegistry: mirror.example.com
     upstream:
       backstage:
         image:
           registry: mirror.example.com
           repository: rhdh/rhdh-hub-rhel9
           tag: sha256:abc...
     EOF
     ```
   
   - Update all platform-specific procedures (OpenShift, AKS, EKS, GKE) with manual examples
   - Align LightSpeed examples with current [chart structure](https://github.com/redhat-developer/rhdh-chart):
     - Replace `global.lightspeed.*` with `intelligentAssistant.*`
     - Show manual extraction and override using `yq` and custom values
   - Document plugin mirroring with manual `skopeo` commands:
     ```bash
     # Extract plugin catalog index
     skopeo copy oci://registry.access.redhat.com/rhdh/plugin-catalog-index:2.1.0 \
       oci://mirror.example.com/rhdh/plugin-catalog-index:2.1.0
     
     # Extract plugin package references from index, then mirror each
     ```
   - **Optional Automation**:
     > The operator repository provides [`mirror-plugins.sh`](https://github.com/redhat-developer/rhdh-operator/blob/main/.rhdh/scripts/mirror-plugins.sh) which automates plugin catalog mirroring. Review the script source before running in production environments.

3. **LightSpeed Air-Gap Documentation** ([current docs](https://docs.redhat.com/en/documentation/red_hat_developer_hub/1.10/html/interacting_with_red_hat_developer_lightspeed_for_red_hat_developer_hub/configure-to-initialize-the-ai-assistant_interacting-with-developer-lightspeed-for-rhdh#mirror-images-for-air-gapped-environments_configure-to-initialize-the-ai-assistant)):
   - Remove manual bundle extraction requirement statement
   - Update to reference standard operator/Helm mirror flows
   - Align image references with current chart/operator structure

4. **Orchestrator Air-Gap Documentation** ([current docs](https://docs.redhat.com/en/documentation/red_hat_developer_hub/1.10/html/orchestrator_in_red_hat_developer_hub/install-orchestrator-plugin-in-an-air-gapped-environment-with-the-operator_orchestrator-in-rhdh)):
   - Clarify orchestrator images vs. serverless operator prerequisites
   - Link to [OpenShift Serverless documentation](https://docs.openshift.com/serverless/) for prerequisite operator installation
   - Update `ImageSetConfiguration` examples to include orchestrator images in `additionalImages`

**Cross-linking Strategy**:
- Shared procedures module for common [OC mirror patterns](https://docs.redhat.com/en/documentation/openshift_container_platform/4.17/html-single/disconnected_environments/index)
- Platform-specific notes for [IDMS/ITMS](https://docs.openshift.com/container-platform/4.14/openshift_images/image-configuration.html) vs. Subscription overrides
- Consistent linking between Operator and Helm procedures
- Link to source repositories for developer/contributor documentation:
  - [rhdh-operator repository](https://github.com/redhat-developer/rhdh-operator)
  - [rhdh-chart repository](https://github.com/redhat-developer/rhdh-chart)

### Version-Specific Notes

**Documentation Versioning**:
- Updates apply to RHDH 2.1+ (when relatedImages extension ships)
- Maintain legacy documentation for RHDH 1.x (current manual approach)
- Version switcher should clearly indicate air-gap approach differences

## References

### OpenShift & OLM Documentation

- [OpenShift Container Platform - Disconnected Environments (4.17)](https://docs.redhat.com/en/documentation/openshift_container_platform/4.17/html-single/disconnected_environments/index)
- [OpenShift - OLM in Restricted Networks (4.14)](https://docs.openshift.com/container-platform/4.14/operators/admin/olm-restricted-networks.html)
- [OpenShift - Installing OC Mirror Plugin](https://docs.redhat.com/en/documentation/openshift_container_platform/4.17/html-single/disconnected_environments/index#installation-oc-mirror-installing-plugin_about-installing-oc-mirror-v2)
- [Operator Lifecycle Manager Documentation](https://olm.operatorframework.io/)
- [OLM - Installing Operators](https://olm.operatorframework.io/docs/tasks/install-operator-with-olm)

### Operator SDK Documentation

- [Operator SDK - ClusterServiceVersion Generation](https://sdk.operatorframework.io/docs/olm-integration/generation/#clusterserviceversion-manifests)
- [Operator SDK - API Markers Reference](https://sdk.operatorframework.io/docs/building-operators/golang/references/markers/)
- [Operator SDK - CSV Fields](https://sdk.operatorframework.io/docs/olm-integration/generation/#csv-fields)

### RHDH Product Documentation

- [RHDH 1.10 - Installing in Air-Gapped Environments (Operator)](https://docs.redhat.com/en/documentation/red_hat_developer_hub/1.10/html-single/installing_red_hat_developer_hub_in_an_air-gapped_environment/index#install-rhdh-in-an-air-gapped-environment-with-the-operator_installing-rhdh-in-an-air-gapped-environment)
- [RHDH 1.10 - Installing in Air-Gapped Environments (Helm)](https://docs.redhat.com/en/documentation/red_hat_developer_hub/1.10/html-single/installing_red_hat_developer_hub_in_an_air-gapped_environment/index#install-rhdh-on-ocp-in-an-air-gapped-environment-with-the-helm-chart_installing-rhdh-in-an-air-gapped-environment)
- [RHDH 1.10 - LightSpeed Air-Gap Configuration](https://docs.redhat.com/en/documentation/red_hat_developer_hub/1.10/html/interacting_with_red_hat_developer_lightspeed_for_red_hat_developer_hub/configure-to-initialize-the-ai-assistant_interacting-with-developer-lightspeed-for-rhdh#mirror-images-for-air-gapped-environments_configure-to-initialize-the-ai-assistant)
- [RHDH 1.10 - Orchestrator Air-Gap Installation](https://docs.redhat.com/en/documentation/red_hat_developer_hub/1.10/html/orchestrator_in_red_hat_developer_hub/install-orchestrator-plugin-in-an-air-gapped-environment-with-the-operator_orchestrator-in-rhdh)

### RHDH Source Repository Documentation

- [rhdh-operator - Air-Gap Deployment Guide](https://github.com/redhat-developer/rhdh-operator/blob/main/.rhdh/docs/airgap.adoc)
- [rhdh-operator - Administrator Guide (Subscription Configuration)](https://github.com/redhat-developer/rhdh-operator/blob/main/docs/admin.md#airgapped-environment)
- [rhdh-operator - prepare-restricted-environment.sh](https://github.com/redhat-developer/rhdh-operator/blob/main/.rhdh/scripts/prepare-restricted-environment.sh)

### Team Discussion Context

- RHDH Install Team Refinement Notes (October 8, 2026)
- Feature: [RHDHPLAN-1887 - Centralize air-gap image inventory](https://redhat.atlassian.net/browse/RHDHPLAN-1887)
