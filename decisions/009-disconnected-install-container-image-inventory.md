# ADR-009: Disconnected Install Container Image Inventory (Operator)

## Executive Summary / TL;DR

Extend RHDH Operator's CSV [spec.relatedImages](https://github.com/redhat-developer/rhdh-operator/blob/main/bundle/rhdh/manifests/backstage-operator.clusterserviceversion.yaml) to include all static container images (LightSpeed/Intelligent Assistant). Helm chart documentation is already aligned with [values.yaml](https://github.com/redhat-developer/rhdh-chart/blob/main/charts/redhat-developer-hub/values.yaml) as the image inventory source. For example throughout helm docs we recommend `helm show values` to identify the images, see [Mirror Developer Lightspeed for RHDH images for Helm deployments on OpenShift Container Platform](https://docs.redhat.com/en/documentation/red_hat_developer_hub/1.10/html/interacting_with_red_hat_developer_lightspeed_for_red_hat_developer_hub/configure-to-initialize-the-ai-assistant_interacting-with-developer-lightspeed-for-rhdh#mirror-images-for-helm-deployments-on-ocp_mirror-images-for-air-gapped-environments), etc. This ADR primarily focused on the operator unless otherwise specified. 

**Why**: Compliance with [OpenShift disconnected operator requirements](https://docs.redhat.com/en/documentation/openshift_container_platform/4.22/html-single/disconnected_environments/index#olm-restricted-networks). Per OpenShift documentation:

> "While OLM can manage Operators from local sources, the ability for a given Operator to run successfully in a disconnected environment still depends on the Operator itself meeting the following criteria:
>
> - **List any related images**, or other container images that the Operator might require to perform their functions, in the **relatedImages parameter of its ClusterServiceVersion (CSV) object**.
> - **Reference all specified images by a digest (SHA)** and not by a tag."
>
> — [OpenShift Container Platform 4.22 - Using Operator Lifecycle Manager in disconnected environments](https://docs.redhat.com/en/documentation/openshift_container_platform/4.22/html-single/disconnected_environments/index#olm-restricted-networks)

**Current Gap**: RHDH currently lists only 2 images (PostgreSQL, RHDH backend). Default-enabled features like Intelligent Assistant require [manual bundle extraction](https://docs.redhat.com/en/documentation/red_hat_developer_hub/1.10/html/interacting_with_red_hat_developer_lightspeed_for_red_hat_developer_hub/configure-to-initialize-the-ai-assistant_interacting-with-developer-lightspeed-for-rhdh#mirror-images-for-air-gapped-environments_configure-to-initialize-the-ai-assistant), contradicting the OpenShift pattern.

### Images Being Added to `spec.relatedImages`

**Currently in CSV** ([view here](https://github.com/redhat-developer/rhdh-operator/blob/main/bundle/rhdh/manifests/backstage-operator.clusterserviceversion.yaml#L498-L502)):

```bash
quay.io/fedora/postgresql-18:latest              # name: postgresql
quay.io/rhdh-community/rhdh:next                  # name: backstage
```

**Will be added**:

*LightSpeed / Intelligent Assistant:*

```bash
# RAG vector database init container
registry.access.redhat.com/rhdh/rhdh-lightspeed-rag-content-init-rhel9@sha256:...  # name: lightspeed-rag-init

# LightSpeed AI service sidecar
registry.redhat.io/lightspeed-core/lightspeed-stack-rhel9@sha256:...               # name: lightspeed-core

# Offline Knowledge Portal (documentation server for RAG)
registry.redhat.io/offline-knowledge-portal/rhokp-rhel9@sha256:...                 # name: okp
```

*Orchestrator:*

```bash
# No static container images - orchestrator uses plugin OCI packages only
# Plugin packages (NOT container images) handled separately via mirror-plugins.sh
```

**On the orchestrator side of things**, we need only to update the documentation, e.g. [Install Orchestrator plugin in an air-gapped environment with the Operator](https://docs.redhat.com/en/documentation/red_hat_developer_hub/1.10/html/orchestrator_in_red_hat_developer_hub/install-orchestrator-plugin-in-an-air-gapped-environment-with-the-operator_orchestrator-in-rhdh) to:
- Not mention `You have mirrored the Red Hat Developer Hub Operator images to the local registry using the RHDH mirroring script.` since we will be opting for manual airgap step and only mentioning the helper script as a fall back at the end of the docs.
- We need documentation for installing orchestrator in an airgap env on vanilla k8s via helm, manual steps also mentioning the helper script at the end also because we currently only have [Install Red Hat Developer Hub with Orchestrator in a fully disconnected OpenShift Container Platform environment using the Helm chart ](https://docs.redhat.com/en/documentation/red_hat_developer_hub/1.10/html/orchestrator_in_red_hat_developer_hub/install-orchestrator-plugin-in-an-air-gapped-environment-with-the-helm-chart_orchestrator-in-rhdh#install-rhdh-with-orchestrator-in-a-fully-disconnected-ocp-environment-using-the-helm-chart_install-orchestrator-plugin-in-an-air-gapped-environment-with-the-helm-chart) using oc-mirror, we also need k8s using `skopeo copy` and utilizing `mirror-plugins.sh`.


### Key Decisions at a High Level

1. **Operator**: Extend existing `RELATED_IMAGE_`* environment variable pattern to LightSpeed images
2. **Documentation**: Manual-first approach with transparent bash snippets (`yq`, `skopeo`, `oc-mirror`); scripts (`prepare-restricted-environment.sh`, `mirror-plugins.sh`) as optional automation
3. **Separation of concerns**:
  - **Static container images** → `spec.relatedImages` in RHDH CSV
  - **Plugin OCI artifacts** → `mirror-plugins.sh` script (separate flow)
4. **User Override**: Subscription `spec.config.env` (OLM v0) or deployment patch (OLM v1) for edge cases (hosted control planes)
5. **Tooling**: Update `prepare-restricted-environment.sh` to read from `spec.relatedImages` first (backward compatible fallback)

---

## Context

**Problem**: RHDH customers on disconnected or restricted networks cannot reliably discover, mirror, and pull every **static container image** required for a supported install including default-enabled capabilities such as Intelligent Assistant—because there is no single single source of truth.

The RHDH Operator can cause pods to pull images defined not only in the ClusterServiceVersion operands but also in **flavour** and **plugin-deps** bundle ConfigMaps (for example `lightspeed-core`, OKP, etc). Today [spec.relatedImages](https://github.com/redhat-developer/rhdh-operator/blob/main/bundle/rhdh/manifests/backstage-operator.clusterserviceversion.yaml#L498-L502) lists only a subset of those images, and disconnected tooling ([prepare-restricted-environment.sh](https://github.com/redhat-developer/rhdh-operator/blob/main/.rhdh/scripts/prepare-restricted-environment.sh)) does **not** treat `relatedImages` as the mirror source of truth, it discovers images mainly by processing CSV files inside operator bundles.

### OpenShift Disconnected Operator Requirements

Per [OpenShift Container Platform documentation on OLM in disconnected environments](https://docs.redhat.com/en/documentation/openshift_container_platform/4.22/html-single/disconnected_environments/index#olm-restricted-networks), operators must meet the following criteria to run successfully in disconnected environments:

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


## Implementation repositories


| Concern                                              | Owning repository                                                                                                                   | Canonical paths (main branch)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | Planned validation / delivery                                                                           |
| ---------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------- |
| Operator **declare** (source manifests → bundle CSV) | [redhat-developer/rhdh-operator](https://github.com/redhat-developer/rhdh-operator)                                                 | Profile: [config/profile/rhdh/](https://github.com/redhat-developer/rhdh-operator/tree/main/config/profile/rhdh) (flavours, plugin-deps, default-config). Shipped CSV: [bundle/rhdh/manifests/backstage-operator.clusterserviceversion.yaml](https://github.com/redhat-developer/rhdh-operator/blob/main/bundle/rhdh/manifests/backstage-operator.clusterserviceversion.yaml) ([spec.relatedImages](https://github.com/redhat-developer/rhdh-operator/blob/main/bundle/rhdh/manifests/backstage-operator.clusterserviceversion.yaml#L498-L502) + operator deployment `RELATED_IMAGE_`* [env vars](https://github.com/redhat-developer/rhdh-operator/blob/main/bundle/rhdh/manifests/backstage-operator.clusterserviceversion.yaml#L311-L316)). Reconcile substitution: [pkg/model/deployment.go](https://github.com/redhat-developer/rhdh-operator/blob/main/pkg/model/deployment.go#L112-L114), [pkg/model/db-statefulset.go](https://github.com/redhat-developer/rhdh-operator/blob/main/pkg/model/db-statefulset.go#L17) | Existing pattern extends to flavor images                                                               |
| Operator **discover / mirror**                       | [redhat-developer/rhdh-operator](https://github.com/redhat-developer/rhdh-operator)                                                 | [.rhdh/scripts/prepare-restricted-environment.sh](https://github.com/redhat-developer/rhdh-operator/blob/main/.rhdh/scripts/prepare-restricted-environment.sh), [.rhdh/scripts/mirror-plugins.sh](https://github.com/redhat-developer/rhdh-operator/blob/main/.rhdh/scripts/mirror-plugins.sh) (plugins only)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Tooling change: prefer CSV `spec.relatedImages`, fallback to old parsing for backward compatibility     |
| Operator **product docs** (in-repo)                  | [redhat-developer/rhdh-operator](https://github.com/redhat-developer/rhdh-operator)                                                 | `.rhdh/docs/airgap.adoc`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              | Align with inventory-first flow; propagate to customer docs                                             |
| Helm **declare**                                     | [redhat-developer/rhdh-chart](https://github.com/redhat-developer/rhdh-chart)                                                       | [charts/redhat-developer-hub/values.yaml](https://github.com/redhat-developer/rhdh-chart/blob/main/charts/redhat-developer-hub/values.yaml), [README.md](https://github.com/redhat-developer/rhdh-chart/blob/main/charts/redhat-developer-hub/README.md) The only thing that needs to be done for helm is adding upstream documentation re: orch airgap on vanilla k8s                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | Chart release tags bind inventory to a chart version; customers use `helm show values` for that version |
| Customer **mirror / install procedures**             | [redhat-developer/red-hat-developers-documentation-rhdh](https://github.com/redhat-developer/red-hat-developers-documentation-rhdh) | user doc updates: Install rhdh airgap with manual steps+scripts. Install rhdh+IA with manual steps(relatedImages approach) with manual steps+scripts. Install rhdh+orchstrator on vanilla k8s using the helm chart                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | Doc updates only                                                 |




## Downstream Impact

Adding images to the upstream CSV's `spec.relatedImages` has a small but important downstream impact on the midstream build pipeline.

### Background: How Downstream Handles Images

The downstream build process performs two key transformations:

1. **Digest pinning**: Upstream tags (e.g., `:latest`, `:0.7.0`) are automatically converted to digest-pinned product images (e.g., `@sha256:abc123...`) by the Konflux build pipeline.

2. **Staleness detection**: The `sync-midstream.sh` script compares image digests in the bundle against the latest digests in Red Hat's product registry. When digests don't match, it triggers a bundle rebuild to pick up the newer image.

### Current State

Today, LightSpeed and OKP images exist only in ConfigMaps, and `sync-midstream.sh` already checks those ConfigMap entries for staleness. PostgreSQL and Backstage images exist in both ConfigMaps and the CSV's `spec.relatedImages`, so they have checks in both places.

### What Changes

This ADR adds LightSpeed and OKP images to the CSV's `spec.relatedImages`. This means these images will now exist in **two locations**:
- ConfigMaps (already checked)
- CSV `spec.relatedImages` (not yet checked)

### Required Update

The downstream team needs to add staleness checks for the new CSV entries in `sync-midstream.sh`, following the same pattern as PostgreSQL.

**Location**: Add checks around right after the existing OKP ConfigMap check ends

**Images requiring new checks**:
- `lightspeed-core` (LightSpeed AI service sidecar)
- `okp` (Offline Knowledge Portal)  fulfills the TODO comment on `sync-midstream.sh`
- `lightspeed-rag-init` (RAG initialization container, if included)

**Why this matters**: Without these checks, when Red Hat releases new versions of these product images, the ConfigMaps will update automatically but the CSV's `spec.relatedImages` entries will remain outdated. This breaks the air-gap workflow because mirror tooling reads from `spec.relatedImages` to discover which images to copy.

**No changes needed**: The `replaceUpstreamImages.sh` script already handles ConfigMap digest pinning correctly and requires no updates.

## Complete Image Inventory

This section provides the comprehensive list of container images that will be included in `spec.relatedImages`, clarifying what RHDH provides vs. what are external prerequisites.

### Current Images (Already in relatedImages)


| Image Name   | Registry Reference (Upstream Example) | Purpose                                     | CSV Entry                                                                                                                                                |
| ------------ | ------------------------------------- | ------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `postgresql` | `quay.io/fedora/postgresql-18:latest` | Local database (when `enableLocalDb: true`) | [View in CSV](https://github.com/redhat-developer/rhdh-operator/blob/main/bundle/rhdh/manifests/backstage-operator.clusterserviceversion.yaml#L171-L172) |
| `backstage`  | `quay.io/rhdh-community/rhdh:next`    | Main RHDH backend container                 | [View in CSV](https://github.com/redhat-developer/rhdh-operator/blob/main/bundle/rhdh/manifests/backstage-operator.clusterserviceversion.yaml#L173-L174) |


**Current Status**: These 2 images are properly declared in `spec.relatedImages` and handled via `RELATED_IMAGE_`* environment variables ([postgresql](https://github.com/redhat-developer/rhdh-operator/blob/main/bundle/rhdh/manifests/backstage-operator.clusterserviceversion.yaml), [backstage](https://github.com/redhat-developer/rhdh-operator/blob/main/bundle/rhdh/manifests/backstage-operator.clusterserviceversion.yaml)).

### New Images Being Added (LightSpeed / Intelligent Assistant)

**Source**: [LightSpeed flavor ConfigMap](https://github.com/redhat-developer/rhdh-operator/tree/main/config/profile/rhdh/flavour-lightspeed) and [OKP plugin-deps](https://github.com/redhat-developer/rhdh-operator/blob/main/config/profile/rhdh/plugin-deps/okp.yaml)


| Image Name            | Registry Reference (Downstream Example)                                             | Purpose                                             | Will Add To                                                        |
| --------------------- | ----------------------------------------------------------------------------------- | --------------------------------------------------- | ------------------------------------------------------------------ |
| `lightspeed-rag-init` | `registry.access.redhat.com/rhdh/rhdh-lightspeed-rag-content-init-rhel9@sha256:...` | LightSpeed RAG data initialization (init container) | `spec.relatedImages` + `RELATED_IMAGE_lightspeed_rag_init` env var |
| `lightspeed-core`     | `registry.redhat.io/lightspeed-core/lightspeed-stack-rhel9@sha256:...`              | LightSpeed backend service (sidecar container)      | `spec.relatedImages` + `RELATED_IMAGE_lightspeed_core` env var     |
| `okp`                 | `registry.redhat.io/offline-knowledge-portal/rhokp-rhel9@sha256:...`                | Offline Knowledge Portal (Solr + httpd for RAG)     | `spec.relatedImages` + `RELATED_IMAGE_okp` env var                 |


### Orchestrator Infrastructure Images

The orchestrator feature primarily uses **dynamic plugin OCI artifacts** (plugin packages that get extracted by the plugin-installer, not run as containers). These will be handled through dedicated tooling and documentation:

1. **Update Orchestrator Air-Gap Documentation** ([current docs](https://docs.redhat.com/en/documentation/red_hat_developer_hub/1.10/html/orchestrator_in_red_hat_developer_hub/install-orchestrator-plugin-in-an-air-gapped-environment-with-the-operator_orchestrator-in-rhdh)) to clearly distinguish Prerequisites since we will not be initially recommending the helper script.
2. **Manual-First Documentation Approach on vanilla K8s**:
  ```bash
   # Mirror orchestrator plugin packages manually
   # Extract plugin references from catalog index, then mirror each package
   skopeo copy \
     oci://registry.access.redhat.com/rhdh/red-hat-developer-hub-backstage-plugin-orchestrator@sha256:... \
     oci://mirror.example.com/rhdh/red-hat-developer-hub-backstage-plugin-orchestrator@sha256:...

   # Repeat for orchestrator-backend, scaffolder-backend-module-orchestrator, etc.
  ```
  ending the docs with an Optional Automation: Link to [mirror-plugins.sh](https://github.com/redhat-developer/rhdh-operator/blob/main/.rhdh/scripts/mirror-plugins.sh) as optional automation for users who prefer it, with clear notice to review script source before running.

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

## Decision

Adopt a **Declare → Discover → Mirror → Resolve** model for disconnected **static container images** in the Operator:


| Stage        | Operator (OLM)                                                                                                                                                                                                                                             |
| ------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Declare**  | Complete CSV `spec.relatedImages` (digest-pinned where practical) + `RELATED_IMAGE_`* environment variables in operator deployment, including all operator-caused static images (flavours, plugin-deps, operands). Extends existing pattern. |
| **Discover** | Supported mirror tooling reads `spec.relatedImages` first, falls back to ad-hoc CSV parsing for backward compatibility with older bundles                                                                                                                  |
| **Mirror**   | Copy declared/discovered images to the customer registry via `prepare-restricted-environment.sh` / `oc-mirror` flows                                                                                                                                       |
| **Resolve**  | Prefer platform mirror configuration (IDMS/ITMS) where supported; users can override via Subscription `spec.config.env` for RELATED_IMAGE vars (OLM v0) or equivalent ClusterExtension configuration (OLM v1)                                              |

**Note**: For Helm, the chart itself requires no changes. Chart `values.yaml` defaults are already the authoritative inventory and documentation already instructs users to use `helm show values` to discover images.

### Operator RELATED_IMAGE Environment Variable Pattern

The operator **already implements** the OpenShift disconnected operator pattern for core images (PostgreSQL, Backstage). This ADR **extends** that pattern to flavor and optional component images.

#### How It Works

1. **CSV Declaration** - Both [spec.relatedImages](https://github.com/redhat-developer/rhdh-operator/blob/main/bundle/rhdh/manifests/backstage-operator.clusterserviceversion.yaml#L498-L502) and [operator deployment env vars](https://github.com/redhat-developer/rhdh-operator/blob/main/bundle/rhdh/manifests/backstage-operator.clusterserviceversion.yaml#L311-L316):

```yaml
# bundle/rhdh/manifests/backstage-operator.clusterserviceversion.yaml
apiVersion: operators.coreos.com/v1alpha1
kind: ClusterServiceVersion
spec:
  relatedImages:
  - image: registry.access.redhat.com/rhdh/rhdh-hub-rhel9@sha256:abc123...
    name: backstage
  - image: registry.access.redhat.com/rhdh/rhdh-rhel9-postgresql-15@sha256:def456...
    name: postgresql
  - image: registry.access.redhat.com/rhdh/rhdh-lightspeed-rag-content-init-rhel9@sha256:ghi789...
    name: lightspeed-rag-init
  - image: registry.redhat.io/lightspeed-core/lightspeed-stack-rhel9@sha256:jkl012...
    name: lightspeed-core
  - image: registry.redhat.io/offline-knowledge-portal/rhokp-rhel9@sha256:mno345...
    name: okp
  
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
                  value: registry.access.redhat.com/rhdh/rhdh-hub-rhel9@sha256:abc123...
                - name: RELATED_IMAGE_postgresql
                  value: registry.access.redhat.com/rhdh/rhdh-rhel9-postgresql-15@sha256:def456...
                - name: RELATED_IMAGE_lightspeed_rag_init
                  value: registry.access.redhat.com/rhdh/rhdh-lightspeed-rag-content-init-rhel9@sha256:ghi789...
                - name: RELATED_IMAGE_lightspeed_core
                  value: registry.redhat.io/lightspeed-core/lightspeed-stack-rhel9@sha256:jkl012...
                - name: RELATED_IMAGE_okp
                  value: registry.redhat.io/offline-knowledge-portal/rhokp-rhel9@sha256:mno345...
```

1. **Operator Code** - Reads environment variables at runtime (existing pattern in [pkg/model/deployment.go](https://github.com/redhat-developer/rhdh-operator/blob/main/pkg/model/deployment.go#L31) and [pkg/model/db-statefulset.go](https://github.com/redhat-developer/rhdh-operator/blob/main/pkg/model/db-statefulset.go#L17)):

```go
// pkg/model/deployment.go
const BackstageImageEnvVar = "RELATED_IMAGE_backstage"

// At reconciliation time:
if os.Getenv(BackstageImageEnvVar) != "" {
    b.setImage(ptr.To(os.Getenv(BackstageImageEnvVar)), i)
}
```

**User Override** - Via [Subscription](https://github.com/redhat-developer/rhdh-operator/blob/main/config/samples/catalog-subscription-template.yaml) (OLM v0) or [ClusterExtension](https://github.com/redhat-developer/rhdh-operator/blob/main/config/samples/catalog-cluster-extension-template.yaml) (OLM v1), we should document this at least in the upstream rhdh-operator docs

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
      - name: RELATED_IMAGE_lightspeed_rag_init
        value: mirror.example.com/rhdh/rhdh-lightspeed-rag-content-init-rhel9@sha256:ghi789...
      - name: RELATED_IMAGE_lightspeed_core
        value: mirror.example.com/lightspeed-core/lightspeed-stack-rhel9@sha256:jkl012...
      - name: RELATED_IMAGE_okp
        value: mirror.example.com/offline-knowledge-portal/rhokp-rhel9@sha256:mno345...
```

**OLM v1 - ClusterExtension with Environment Variable Override:**

In OLM v1, environment variable overrides may be applied differently and because OLM v1 architecture is evolving. The preferred method for environment variable overrides in ClusterExtension may change. 

**Implementation approach**:

**Operator**

- Extend the existing `RELATED_IMAGE_*` [pattern](https://github.com/redhat-developer/rhdh-operator/blob/main/bundle/rhdh/manifests/backstage-operator.clusterserviceversion.yaml#L311-L316) to include:
  - **LightSpeed/Intelligent Assistant images** (3 container images):
    - `lightspeed-rag-init` - RAG vector database init container ([flavour ConfigMap](https://github.com/redhat-developer/rhdh-operator/tree/main/config/profile/rhdh/flavour-lightspeed))
    - `lightspeed-core` - LightSpeed AI service sidecar
    - `okp` - Offline Knowledge Portal ([plugin-deps ConfigMap](https://github.com/redhat-developer/rhdh-operator/blob/main/config/profile/rhdh/plugin-deps/okp.yaml))
  - **Orchestrator**: Documentation only
- Update **operator air-gap documentation** (`.rhdh/docs/airgap.adoc` and downstream product docs) with **manual-first approach**:
  - **Primary documentation**: Step-by-step manual commands using standard tooling:
    - Extract image list from CSV `spec.relatedImages` using `yq` or `jq`
    - Mirror images using standard tools (`oc-mirror` v2 on OpenShift, `skopeo` on Kubernetes)
    - Apply IDMS/ITMS or configure Subscription overrides
    - Clear, transparent bash snippets (10-20 lines max) that users can understand and customize
  - **Secondary option**: Link to [prepare-restricted-environment.sh](https://github.com/redhat-developer/rhdh-operator/blob/main/.rhdh/scripts/prepare-restricted-environment.sh) as optional automation for those who prefer it
  - **Rationale**: Enterprise customers often have security policies against running large external scripts; they prefer transparent, auditable commands they control
  - Document the Subscription `spec.config.env` override pattern with examples
  - Reference OLM v1 ClusterExtension equivalent configuration
  - Align LightSpeed documentation with `relatedImages` approach (remove manual extraction steps)
- Update **disconnected tooling** ([prepare-restricted-environment.sh](https://github.com/redhat-developer/rhdh-operator/blob/main/.rhdh/scripts/prepare-restricted-environment.sh)) to:
  - Read from [spec.relatedImages](https://github.com/redhat-developer/rhdh-operator/blob/main/bundle/rhdh/manifests/backstage-operator.clusterserviceversion.yaml#L498-L502) as primary source
  - Maintain fallback to old CSV parsing for backward compatibility with previous install plans
  - Keep `--extra-images` for site-specific exceptions not undeclared product images
  - **Position in docs**: Optional automation layer, not the primary procedure
- **Note**: Operator reconcile already handles environment variable substitution for core images; extending to additional images uses the same code paths.



**Helm**

- Orchestrator documentation updates only. Essentially to install orchestrator on vanilla k8s using skopeo commands and add optional `mirror-plugins.sh`.

## Alternatives Considered

### Alternative 1: Document-only fixes (`--extra-images`, manual bundle extract)

- **Approach**: Leave `relatedImages` and scripts unchanged; extend runbooks with per-image manual steps.
- **Rejected because**: Does not scale with flavours and plugin-deps; guarantees recurring gaps; contradicts OpenShift disconnected operator guidance.



### Alternative 2: Scan all bundle YAML for `image:` instead of `relatedImages`

- **Approach**: Teach the mirror script to walk every bundle manifest.
- **Rejected because**: Duplicates OLM's intended contract, is fragile across packaging changes, and does not give customers a single CSV field to audit; may be used as a **temporary** safety net in tooling but not as the primary design.



### Alternative 3: Operator substitutes every image in reconciled manifests

- **Approach**: Immediately wire all sidecars and deps through `RELATED_IMAGE_`* env vars.
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

## References

### OpenShift & OLM Documentation

- [OpenShift Container Platform - Disconnected Environments (4.17)](https://docs.redhat.com/en/documentation/openshift_container_platform/4.17/html-single/disconnected_environments/index)
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
- [rhdh-operator - mirror-plugins.sh](https://github.com/redhat-developer/rhdh-operator/blob/main/.rhdh/scripts/mirror-plugins.sh)

### Team Discussion Context

- RHDH Install Team Refinement Notes (October 8, 2026)
- Feature: [RHDHPLAN-1887 - Centralize air-gap image inventory](https://redhat.atlassian.net/browse/RHDHPLAN-1887)
