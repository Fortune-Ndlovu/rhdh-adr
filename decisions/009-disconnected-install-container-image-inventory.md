# ADR-009: Disconnected Install Container Image Inventory (Operator and Helm)

## Executive Summary / TL;DR

**What**: Extend RHDH Operator's CSV `[spec.relatedImages](https://github.com/redhat-developer/rhdh-operator/blob/main/bundle/rhdh/manifests/backstage-operator.clusterserviceversion.yaml)` to include all static container images (LightSpeed/Intelligent Assistant) and align Helm chart documentation with `[values.yaml](https://github.com/redhat-developer/rhdh-chart/blob/main/charts/redhat-developer-hub/values.yaml)` as the image inventory source.

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

**NOT in** `relatedImages` (handled separately):

*Plugin OCI Artifacts* (mirrored via `[mirror-plugins.sh](https://github.com/redhat-developer/rhdh-operator/blob/main/.rhdh/scripts/mirror-plugins.sh)`):

```bash
registry.access.redhat.com/rhdh/red-hat-developer-hub-backstage-plugin-orchestrator@sha256:...
registry.access.redhat.com/rhdh/red-hat-developer-hub-backstage-plugin-orchestrator-backend@sha256:...
# ...more plugin packages (these are OCI artifacts, not container images)
```

current airgapping docs recommend using oc-mirror on ocp which is perfect. We may need to extend it for vanilla k8s which will all us to utlize the mirror-plugins.sh and manual extraction process, [https://docs.redhat.com/en/documentation/red_hat_developer_hub/1.10/html/orchestrator_in_red_hat_developer_hub/install-orchestrator-plugin-in-an-air-gapped-environment-with-the-operator_orchestrator-in-rhdh#install-rhdh-with-orchestrator-in-a-fully-disconnected-ocp-environment-using-the-operator_install-orchestrator-plugin-in-an-air-gapped-environment-with-the-operator](https://docs.redhat.com/en/documentation/red_hat_developer_hub/1.10/html/orchestrator_in_red_hat_developer_hub/install-orchestrator-plugin-in-an-air-gapped-environment-with-the-operator_orchestrator-in-rhdh#install-rhdh-with-orchestrator-in-a-fully-disconnected-ocp-environment-using-the-operator_install-orchestrator-plugin-in-an-air-gapped-environment-with-the-operator)

### Key Decisions

1. **Operator**: Extend existing `RELATED_IMAGE_`* environment variable pattern to LightSpeed images
2. **Documentation**: Manual-first approach with transparent bash snippets (`yq`, `skopeo`, `oc-mirror`); scripts (`prepare-restricted-environment.sh`, `mirror-plugins.sh`) as optional automation
3. **Separation of concerns**:
  - **Static container images** → `spec.relatedImages` in RHDH CSV (this ADR)
  - **Plugin OCI artifacts** → `mirror-plugins.sh` script (separate flow)
4. **User Override**: Subscription `spec.config.env` (OLM v0) or deployment patch (OLM v1) for edge cases (hosted control planes)
5. **Tooling**: Update `prepare-restricted-environment.sh` to read from `spec.relatedImages` first (backward compatible fallback)
6. **Helm**: Align docs with chart `values.yaml` structure; no new artifacts needed

---



## Context

**Problem**: RHDH customers on disconnected or restricted networks cannot reliably discover, mirror, and pull every **static container image** required for a supported install—including default-enabled capabilities such as Intelligent Assistant—because there is no single single source of truth.

The RHDH Operator can cause pods to pull images defined not only in the ClusterServiceVersion operands but also in **flavour** and **plugin-deps** bundle ConfigMaps (for example `lightspeed-core`, OKP, Tekton UBI, Chainguard git). Today `[spec.relatedImages](https://github.com/redhat-developer/rhdh-operator/blob/main/bundle/rhdh/manifests/backstage-operator.clusterserviceversion.yaml#L498-L502)` lists only a subset of those images, and disconnected tooling (`[prepare-restricted-environment.sh](https://github.com/redhat-developer/rhdh-operator/blob/main/.rhdh/scripts/prepare-restricted-environment.sh)`) does **not** treat `relatedImages` as the mirror source of truth—it discovers images mainly by processing CSV files inside operator bundles.

**Helm** installs use a different mechanism but the same customer need: every image referenced by chart defaults must be mirrorable and overridable. The `rhdh` chart already properly documents this contract in:

- **Chart values**: `[charts/backstage/values.yaml](https://github.com/redhat-developer/rhdh-chart/blob/main/charts/backstage/values.yaml)` (comprehensive image definitions with registry, repository, tag fields)
- **Chart documentation**: `[docs/intelligent-assistant.md](https://github.com/redhat-developer/rhdh-chart/blob/main/docs/intelligent-assistant.md)` (13KB guide covering LightSpeed/OKP image configuration, including `intelligentAssistant.initContainer.image`, `intelligentAssistant.sidecar.image`, `intelligentAssistant.okp.image.`*)
- **Image override mechanisms**: `global.imageRegistry` for bulk overrides, plus per-component `image.registry/repository/tag` structure

However, product documentation is misaligned with this current chart structure:

**Specific Documentation Gaps (Helm):**

> **Note**: RHDH 1.10 dedicated air-gapped installation book not yet published as of this ADR. References use RHDH 1.7 as latest available showing the pattern. These issues persist across versions and will need fixing when 1.10 air-gap docs publish.

1. **Air-Gapped Helm Installation** ([Reference: RHDH 1.7 example](https://docs.redhat.com/en/documentation/red_hat_developer_hub/1.7/html/installing_red_hat_developer_hub_in_an_air-gapped_environment/assembly-install-rhdh-airgapped-environment-k8s-helm_title-install-rhdh-air-grapped)):
  - **Issue**: Does not instruct users to run `helm show values` to discover the image list from chart defaults
  - **Fix Needed**: Add clear step-by-step manual commands showing how to extract image references from `values.yaml` using `helm show values` + `yq`
  - **Applies to**: All platform-specific air-gap Helm procedures (OpenShift, AKS, EKS, GKE) when 1.10 air-gap book publishes
2. **[RHDH 1.10 - LightSpeed Air-Gap Configuration](https://docs.redhat.com/en/documentation/red_hat_developer_hub/1.10/html-single/interacting_with_red_hat_developer_lightspeed_for_red_hat_developer_hub/index)** (search page for "mirror"):
  - **Issue**: References deprecated field names (`global.lightspeed.`*) that don't exist in current chart; actual fields are documented in chart `[docs/intelligent-assistant.md](https://github.com/redhat-developer/rhdh-chart/blob/main/docs/intelligent-assistant.md)` and `[values.yaml](https://github.com/redhat-developer/rhdh-chart/blob/main/charts/backstage/values.yaml)`
  - **Correct structure**: `intelligentAssistant.initContainer.image.`*, `intelligentAssistant.sidecar.image.*` (aka lightspeed-core), `intelligentAssistant.okp.image.*`
  - **Fix Needed**: Update product docs to reference current chart structure; leverage the comprehensive guide already in chart `[docs/intelligent-assistant.md](https://github.com/redhat-developer/rhdh-chart/blob/main/docs/intelligent-assistant.md)`
  - **Verified working URL**: [https://docs.redhat.com/en/documentation/red_hat_developer_hub/1.10/html-single/interacting_with_red_hat_developer_lightspeed_for_red_hat_developer_hub/index](https://docs.redhat.com/en/documentation/red_hat_developer_hub/1.10/html-single/interacting_with_red_hat_developer_lightspeed_for_red_hat_developer_hub/index)
3. **General Air-Gap Helm Procedures** (all versions):
  - **Issue**: No clear guidance that chart `values.yaml` is the authoritative image inventory (unlike Operator CSV which is being standardized)
  - **Fix Needed**: Document that `helm show values openshift-charts/redhat-developer-hub --version X.Y.Z` is the source of truth for discovering which images to mirror



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

**Constraints and scope**:

- **Inclusion rule (Operator)**: If the Operator can cause a static container image to run as part of a supported deployment (including default-enabled flavours and optional add-ons when enabled), it belongs in the disconnected inventory.



## Implementation repositories


| Concern                                              | Owning repository                                                                                                                   | Canonical paths (main branch)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | Planned validation / delivery                                                                           |
| ---------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------- |
| Operator **declare** (source manifests → bundle CSV) | [redhat-developer/rhdh-operator](https://github.com/redhat-developer/rhdh-operator)                                                 | Profile: `[config/profile/rhdh/](https://github.com/redhat-developer/rhdh-operator/tree/main/config/profile/rhdh)` (flavours, plugin-deps, default-config). Shipped CSV: `[bundle/rhdh/manifests/backstage-operator.clusterserviceversion.yaml](https://github.com/redhat-developer/rhdh-operator/blob/main/bundle/rhdh/manifests/backstage-operator.clusterserviceversion.yaml)` (`[spec.relatedImages](https://github.com/redhat-developer/rhdh-operator/blob/main/bundle/rhdh/manifests/backstage-operator.clusterserviceversion.yaml#L498-L502)` + operator deployment `RELATED_IMAGE_`* [env vars](https://github.com/redhat-developer/rhdh-operator/blob/main/bundle/rhdh/manifests/backstage-operator.clusterserviceversion.yaml#L311-L316)). Reconcile substitution: `[pkg/model/deployment.go](https://github.com/redhat-developer/rhdh-operator/blob/main/pkg/model/deployment.go#L112-L114)`, `[pkg/model/db-statefulset.go](https://github.com/redhat-developer/rhdh-operator/blob/main/pkg/model/db-statefulset.go#L17)` | Existing pattern extends to flavor images                                                               |
| Operator **discover / mirror**                       | [redhat-developer/rhdh-operator](https://github.com/redhat-developer/rhdh-operator)                                                 | `[.rhdh/scripts/prepare-restricted-environment.sh](https://github.com/redhat-developer/rhdh-operator/blob/main/.rhdh/scripts/prepare-restricted-environment.sh)`, `[.rhdh/scripts/mirror-plugins.sh](https://github.com/redhat-developer/rhdh-operator/blob/main/.rhdh/scripts/mirror-plugins.sh)` (plugins only)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Tooling change: prefer CSV `spec.relatedImages`, fallback to old parsing for backward compatibility     |
| Operator **product docs** (in-repo)                  | [redhat-developer/rhdh-operator](https://github.com/redhat-developer/rhdh-operator)                                                 | `.rhdh/docs/airgap.adoc`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              | Align with inventory-first flow; propagate to customer docs                                             |
| Helm **declare**                                     | [redhat-developer/rhdh-chart](https://github.com/redhat-developer/rhdh-chart)                                                       | `[charts/redhat-developer-hub/values.yaml](https://github.com/redhat-developer/rhdh-chart/blob/main/charts/redhat-developer-hub/values.yaml)`, `[README.md](https://github.com/redhat-developer/rhdh-chart/blob/main/charts/redhat-developer-hub/README.md)`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | Chart release tags bind inventory to a chart version; customers use `helm show values` for that version |
| Customer **mirror / install procedures**             | [redhat-developer/red-hat-developers-documentation-rhdh](https://github.com/redhat-developer/red-hat-developers-documentation-rhdh) | Shared modules: `assemblies/modules/shared/proc-mirror-images-for-helm-deployments-on-*.adoc`, `proc-mirror-images-for-operator-deployments.adoc`; air-gap book: `assemblies/modules/install_installing-rhdh-in-an-air-gapped-environment/`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | Doc updates only (no second inventory artifact for Helm)                                                |




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


**Helm Equivalent**: 

- RAG init: `[intelligentAssistant.initContainer.image](https://github.com/redhat-developer/rhdh-chart/blob/main/charts/redhat-developer-hub/values.yaml)`
- LightSpeed core: `[intelligentAssistant.sidecar.image](https://github.com/redhat-developer/rhdh-chart/blob/main/charts/redhat-developer-hub/values.yaml)`
- OKP: `[intelligentAssistant.okp.image](https://github.com/redhat-developer/rhdh-chart/blob/main/charts/redhat-developer-hub/values.yaml)`

**User Action**: Mirror all three images alongside core RHDH images. No separate prerequisite installation required.

### Orchestrator Infrastructure Images

**Important Distinction**: 

- **Container Images** (in scope for this ADR) - static containers RHDH operator deploys
- **Plugin OCI Artifacts** (out of scope) - handled by `[mirror-plugins.sh](https://github.com/redhat-developer/rhdh-operator/blob/main/.rhdh/scripts/mirror-plugins.sh)`

**Decision: Plugin OCI Artifacts Cannot Go in** `spec.relatedImages`

Based on [OpenShift OLM disconnected operator requirements](https://docs.redhat.com/en/documentation/openshift_container_platform/4.4/html/operators/olm-restricted-networks) and the [OLM operator bundle specification](https://github.com/openshift/enhancements/blob/master/enhancements/olm/operator-bundle.md), the `spec.relatedImages` field is strictly for **runnable container images only**, not non-container OCI artifacts:

> "List any related images, or other **container images** that the Operator might require to perform their functions, in the relatedImages parameter of its ClusterServiceVersion (CSV) object."
>
> — [OpenShift Container Platform 4.4+ - Using OLM in Disconnected Environments](https://docs.redhat.com/en/documentation/openshift_container_platform/4.4/html/operators/olm-restricted-networks)

**Why Plugin Packages Don't Belong in relatedImages:**

1. **Semantic Contract**: The [OLM ClusterServiceVersion specification](https://olm.operatorframework.io/docs/concepts/crds/clusterserviceversion/) defines `relatedImages` as containing "all **container images** required by the Operator" - consistently referring to executable images, not generic OCI artifacts.
2. **Mirroring Tools Expect Container Images**: `oc-mirror` processes `relatedImages` entries as container images for `ImageDigestMirrorSet` (IDMS) configuration. Per [OKD OLM restricted networks documentation](https://docs.okd.io/4.7/operators/admin/olm-restricted-networks.html), IDMS "tells the cluster to look for images in your mirror registry" when **pods pull** them. Plugin packages are never pulled by pods as container images.
3. **No Real-World Examples**: All surveyed OpenShift operators ([OpenTelemetry](https://github.com/openshift/open-telemetry-opentelemetry-operator/pull/246), [Lightspeed](https://github.com/openshift/lightspeed-operator/pull/2083), [SPIRE](https://github.com/openshift/zero-trust-workload-identity-manager/pull/180), [Kueue](https://github.com/openshift/kueue-operator/pull/2599)) list only runnable container images (operator controllers, operands, sidecars, must-gather) in `relatedImages` — zero examples of non-container OCI artifacts like Helm charts, WASM modules, or plugin packages.

**What We'll Do Instead - Orchestrator Plugin Mirroring:**

The orchestrator feature primarily uses **dynamic plugin OCI artifacts** (plugin packages that get extracted by the plugin-installer, not run as containers). These will be handled through dedicated tooling and documentation:

1. **Update Orchestrator Air-Gap Documentation** ([current docs](https://docs.redhat.com/en/documentation/red_hat_developer_hub/1.10/html/orchestrator_in_red_hat_developer_hub/install-orchestrator-plugin-in-an-air-gapped-environment-with-the-operator_orchestrator-in-rhdh)) to clearly distinguish:
  - **Plugin packages** → Mirror using `[mirror-plugins.sh](https://github.com/redhat-developer/rhdh-operator/blob/main/.rhdh/scripts/mirror-plugins.sh)` or manual `skopeo` commands
  - **Serverless operator prerequisites** → Mirror via standard OLM catalog mirroring (they have their own `relatedImages`)
  - **Any static container images** (if found during audit) → Add to RHDH CSV `relatedImages`
2. **Manual-First Documentation Approach**:
  ```bash
   # Mirror orchestrator plugin packages manually
   # Extract plugin references from catalog index, then mirror each package
   skopeo copy \
     oci://registry.access.redhat.com/rhdh/red-hat-developer-hub-backstage-plugin-orchestrator@sha256:... \
     oci://mirror.example.com/rhdh/red-hat-developer-hub-backstage-plugin-orchestrator@sha256:...

   # Repeat for orchestrator-backend, scaffolder-backend-module-orchestrator, etc.
  ```
3. **Optional Automation**: Link to `[mirror-plugins.sh](https://github.com/redhat-developer/rhdh-operator/blob/main/.rhdh/scripts/mirror-plugins.sh)` as optional automation for users who prefer it, with clear notice to review script source before running.

**Action Item**: Audit `[config/profile/rhdh/plugin-deps/](https://github.com/redhat-developer/rhdh-operator/tree/main/config/profile/rhdh/plugin-deps)` to confirm orchestrator has no static container images beyond plugin packages. If static container images are found (unlikely based on current architecture), add them to CSV `relatedImages`.

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


| Operator                   | Package Name          | Purpose                       | Mirror Instructions                                                                                                                                                                                                                                              |
| -------------------------- | --------------------- | ----------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| OpenShift Serverless       | `serverless-operator` | Knative Eventing and Serving  | [OpenShift Serverless Documentation](https://docs.openshift.com/serverless/)                                                                                                                                                                                     |
| OpenShift Serverless Logic | `logic-operator`      | Serverless workflow execution | [Included in orchestrator prerequisites](https://docs.redhat.com/en/documentation/red_hat_developer_hub/1.10/html/orchestrator_in_red_hat_developer_hub/install-orchestrator-plugin-in-an-air-gapped-environment-with-the-operator_orchestrator-in-rhdh#L36-L48) |


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



## Decision

Adopt a shared **Declare → Discover → Mirror → Resolve** model for disconnected **static container images**, with install-method-specific sources of truth:


| Stage        | Operator (OLM)                                                                                                                                                                                                                                             | Helm (`redhat-developer-hub` chart)                                                                                   |
| ------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------- |
| **Declare**  | Complete CSV `spec.relatedImages` (digest-pinned where practical) + `RELATED_IMAGE_`* environment variables in operator deployment, including all operator-caused static images (flavours, plugin-deps, operands, orchestrator). Extends existing pattern. | Chart `values.yaml` **defaults** are the inventory; document how to derive the image list (`helm show values`)        |
| **Discover** | Supported mirror tooling reads `spec.relatedImages` first, falls back to ad-hoc CSV parsing for backward compatibility with older bundles                                                                                                                  | Documentation and examples derive mirrors from chart defaults; no parallel "bundle extract" story                     |
| **Mirror**   | Copy declared/discovered images to the customer registry via `prepare-restricted-environment.sh` / `oc-mirror` flows                                                                                                                                       | Mirror images from values; use `global.imageRegistry` and per-key overrides (`intelligentAssistant.core.image`, etc.) |
| **Resolve**  | Prefer platform mirror configuration (IDMS/ITMS) where supported; users can override via Subscription `spec.config.env` for RELATED_IMAGE vars (OLM v0) or equivalent ClusterExtension configuration (OLM v1)                                              | Explicit values overrides and/or cluster registry configuration per platform docs                                     |




### Operator RELATED_IMAGE Environment Variable Pattern

The operator **already implements** the OpenShift disconnected operator pattern for core images (PostgreSQL, Backstage). This ADR **extends** that pattern to flavor and optional component images.

#### How It Works

1. **CSV Declaration** - Both `[spec.relatedImages](https://github.com/redhat-developer/rhdh-operator/blob/main/bundle/rhdh/manifests/backstage-operator.clusterserviceversion.yaml#L498-L502)` and [operator deployment env vars](https://github.com/redhat-developer/rhdh-operator/blob/main/bundle/rhdh/manifests/backstage-operator.clusterserviceversion.yaml#L311-L316):

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

1. **Operator Code** - Reads environment variables at runtime (existing pattern in `[pkg/model/deployment.go](https://github.com/redhat-developer/rhdh-operator/blob/main/pkg/model/deployment.go#L31)` and `[pkg/model/db-statefulset.go](https://github.com/redhat-developer/rhdh-operator/blob/main/pkg/model/db-statefulset.go#L17)`):

```go
// pkg/model/deployment.go
const BackstageImageEnvVar = "RELATED_IMAGE_backstage"

// At reconciliation time:
if os.Getenv(BackstageImageEnvVar) != "" {
    b.setImage(ptr.To(os.Getenv(BackstageImageEnvVar)), i)
}
```

1. **User Override** - Via [Subscription](https://github.com/redhat-developer/rhdh-operator/blob/main/config/samples/catalog-subscription-template.yaml) (OLM v0) or [ClusterExtension](https://github.com/redhat-developer/rhdh-operator/blob/main/config/samples/catalog-cluster-extension-template.yaml) (OLM v1):

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
# Patch operator deployment to override images (example with LightSpeed images)
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
        "name": "RELATED_IMAGE_lightspeed_rag_init",
        "value": "mirror.example.com/rhdh/rhdh-lightspeed-rag-content-init-rhel9@sha256:ghi789..."
      }
    },
    {
      "op": "add",
      "path": "/spec/template/spec/containers/0/env/-",
      "value": {
        "name": "RELATED_IMAGE_lightspeed_core",
        "value": "mirror.example.com/lightspeed-core/lightspeed-stack-rhel9@sha256:jkl012..."
      }
    },
    {
      "op": "add",
      "path": "/spec/template/spec/containers/0/env/-",
      "value": {
        "name": "RELATED_IMAGE_okp",
        "value": "mirror.example.com/offline-knowledge-portal/rhokp-rhel9@sha256:mno345..."
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

- Extend the existing `RELATED_IMAGE_*` [pattern](https://github.com/redhat-developer/rhdh-operator/blob/main/bundle/rhdh/manifests/backstage-operator.clusterserviceversion.yaml#L311-L316) to include:
  - **LightSpeed/Intelligent Assistant images** (3 container images):
    - `lightspeed-rag-init` - RAG vector database init container ([flavour ConfigMap](https://github.com/redhat-developer/rhdh-operator/tree/main/config/profile/rhdh/flavour-lightspeed))
    - `lightspeed-core` - LightSpeed AI service sidecar
    - `okp` - Offline Knowledge Portal ([plugin-deps ConfigMap](https://github.com/redhat-developer/rhdh-operator/blob/main/config/profile/rhdh/plugin-deps/okp.yaml))
  - **Orchestrator**: No static container images (uses plugin OCI packages only; serverless operators are external prerequisites)
  - Additional flavor and plugin-dependency images from `[config/profile/rhdh/](https://github.com/redhat-developer/rhdh-operator/tree/main/config/profile/rhdh)` as discovered during audit
- Update **operator air-gap documentation** (`.rhdh/docs/airgap.adoc` and downstream product docs) with **manual-first approach**:
  - **Primary documentation**: Step-by-step manual commands using standard tooling:
    - Extract image list from CSV `spec.relatedImages` using `yq` or `jq`
    - Mirror images using standard tools (`oc-mirror` v2 on OpenShift, `skopeo` on Kubernetes)
    - Apply IDMS/ITMS or configure Subscription overrides
    - Clear, transparent bash snippets (10-20 lines max) that users can understand and customize
  - **Secondary option**: Link to `[prepare-restricted-environment.sh](https://github.com/redhat-developer/rhdh-operator/blob/main/.rhdh/scripts/prepare-restricted-environment.sh)` as optional automation for those who prefer it
  - **Rationale**: Enterprise customers often have security policies against running large external scripts; they prefer transparent, auditable commands they control
  - Document the Subscription `spec.config.env` override pattern with examples
  - Reference OLM v1 ClusterExtension equivalent configuration
  - Align LightSpeed documentation with `relatedImages` approach (remove manual extraction steps)
- Update **disconnected tooling** (`[prepare-restricted-environment.sh](https://github.com/redhat-developer/rhdh-operator/blob/main/.rhdh/scripts/prepare-restricted-environment.sh)`) to:
  - Read from `[spec.relatedImages](https://github.com/redhat-developer/rhdh-operator/blob/main/bundle/rhdh/manifests/backstage-operator.clusterserviceversion.yaml#L498-L502)` as primary source
  - Maintain fallback to old CSV parsing for backward compatibility with previous install plans
  - Keep `--extra-images` for site-specific exceptions—not undeclared product images
  - **Position in docs**: Optional automation layer, not the primary procedure
- **Note**: Operator reconcile already handles environment variable substitution for core images; extending to additional images uses the same code paths.



**Helm**

- Treat `helm show values` on the shipped chart version as the customer-facing discovery step for static images.
- Align **product documentation** mirror procedures with current [chart structure](https://github.com/redhat-developer/rhdh-chart/blob/main/charts/redhat-developer-hub/values.yaml) using **manual-first approach**:
  - **Primary documentation**: Clear manual steps:
    - Extract image references from chart values using `helm show values` and `yq`
    - Mirror images using standard tools (`oc-mirror` v2 on OpenShift, `skopeo` on Kubernetes)
    - Override images in custom `values.yaml` using standard Helm mechanisms
    - Provide concise, copy-paste-ready command examples
  - **No large scripts required**: Helm chart values already serve as the image inventory; extraction is straightforward
  - Document image override keys:
    - `[intelligentAssistant.initContainer.image.*](https://github.com/redhat-developer/rhdh-chart/blob/main/charts/redhat-developer-hub/values.yaml#L218-L222)`
    - `[intelligentAssistant.sidecar.image.*](https://github.com/redhat-developer/rhdh-chart/blob/main/charts/redhat-developer-hub/values.yaml#L223-L227)`
    - `[global.imageRegistry](https://github.com/redhat-developer/rhdh-chart/blob/main/charts/redhat-developer-hub/values.yaml#L10)` for registry-wide overrides
    - `ref://` plugin overrides (separate from static container images)
- Remove obsolete references:
  - `global.lightspeed.*` (deprecated keys)
  - RAG init container examples not aligned with current chart
  - OCI Lightspeed plugin examples that don't match current defaults
- Cross-link **plugin mirroring** documentation with same manual-first approach:
  - Show manual `skopeo` commands for plugin OCI artifacts
  - Link to `[mirror-plugins.sh](https://github.com/redhat-developer/rhdh-operator/blob/main/.rhdh/scripts/mirror-plugins.sh)` as optional automation
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



## Documentation Update Plan



### Operator Air-Gap Documentation

**File**: `[rhdh-operator/.rhdh/docs/airgap.adoc](https://github.com/redhat-developer/rhdh-operator/blob/main/.rhdh/docs/airgap.adoc)`

**Updates Needed**:

1. Add section explaining `spec.relatedImages` as source of truth
2. Document Subscription `spec.config.env` override pattern with examples (see `[docs/admin.md](https://github.com/redhat-developer/rhdh-operator/blob/main/docs/admin.md#airgapped-environment)`)
3. Add OLM v1 ClusterExtension configuration notes
4. Update LightSpeed section to reference `relatedImages` (remove manual extraction steps)
5. Add orchestrator image handling with link to [OpenShift Serverless documentation](https://docs.openshift.com/serverless/)

**Cross-links**:

- Link to [OpenShift OLM disconnected environments documentation](https://docs.openshift.com/container-platform/4.14/operators/admin/olm-restricted-networks.html)
- Link to [Operator SDK CSV generation documentation](https://sdk.operatorframework.io/docs/olm-integration/generation/#clusterserviceversion-manifests)
- Link to customer documentation (once updated)



### Customer Product Documentation

**Repository**: `[red-hat-developers-documentation-rhdh](https://github.com/redhat-developer/red-hat-developers-documentation-rhdh)`

**Base Documentation**: [Installing RHDH in an Air-Gapped Environment](https://docs.redhat.com/en/documentation/red_hat_developer_hub/1.10/html-single/installing_red_hat_developer_hub_in_an_air-gapped_environment/)

**Updates Needed**:

1. **Operator Installation Procedures**:
  **Target Documentation Page**: 
  - **Note**: Dedicated "Installing RHDH in an Air-Gapped Environment" book not yet published for 1.10
  - **Reference pattern**: [RHDH 1.7 - Air-Gapped Operator Install](https://docs.redhat.com/en/documentation/red_hat_developer_hub/1.7/html/installing_red_hat_developer_hub_in_an_air-gapped_environment/assembly-install-rhdh-airgapped-environment-ocp-operator_title-install-rhdh-air-grapped)
  - **Alternative**: [RHDH 1.10 - Orchestrator Air-Gap with Operator](https://docs.redhat.com/en/documentation/red_hat_developer_hub/1.10/html/orchestrator_in_red_hat_developer_hub/install-orchestrator-plugin-in-an-air-gapped-environment-with-the-operator_orchestrator-in-rhdh) (shows current operator air-gap patterns)
  **Why This Needs Updates**:
  - Currently doesn't clearly state that CSV `spec.relatedImages` is the authoritative source for the complete image list
  - Doesn't provide manual-first examples (over-relies on `prepare-restricted-environment.sh` script)
  - Missing clear `yq`/`jq` examples for extracting images from CSV
  - Doesn't document Subscription `spec.config.env` override mechanism for edge cases
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
    > For users who prefer automation, the operator repository provides `[prepare-restricted-environment.sh](https://github.com/redhat-developer/rhdh-operator/blob/main/.rhdh/scripts/prepare-restricted-environment.sh)` which automates the steps above. This script requires bash, yq, skopeo, and other dependencies. Review the script source before running in production environments.
  - Document Subscription `spec.config.env` override pattern
  - Reference OLM v1 ClusterExtension deployment patch approach
  - Cross-link to [operator repository documentation](https://github.com/redhat-developer/rhdh-operator/blob/main/.rhdh/docs/airgap.adoc)
2. **Helm Installation Procedures**:
  **Target Documentation Pages**:
  - **Note**: Dedicated "Installing RHDH in an Air-Gapped Environment" book not yet published for 1.10
  - **Reference pattern**: [RHDH 1.7 - Kubernetes Air-Gap Helm](https://docs.redhat.com/en/documentation/red_hat_developer_hub/1.7/html/installing_red_hat_developer_hub_in_an_air-gapped_environment/assembly-install-rhdh-airgapped-environment-k8s-helm_title-install-rhdh-air-grapped)
  - **Platform-specific variants** (when 1.10 publishes): OpenShift, AKS, EKS, GKE air-gap Helm procedures
  **Why These Need Updates**:
  - Currently don't instruct users that `helm show values` is the authoritative image inventory
  - Missing clear extraction examples using `yq` to parse chart defaults
  - Don't document the image override mechanism (`global.imageRegistry`, per-component overrides)
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
    > The operator repository provides `[mirror-plugins.sh](https://github.com/redhat-developer/rhdh-operator/blob/main/.rhdh/scripts/mirror-plugins.sh)` which automates plugin catalog mirroring. Review the script source before running in production environments.
3. **LightSpeed Air-Gap Documentation**:
  **Target Documentation Page**: 
  - **Verified URL**: [Interacting with Red Hat Developer Lightspeed for RHDH](https://docs.redhat.com/en/documentation/red_hat_developer_hub/1.10/html-single/interacting_with_red_hat_developer_lightspeed_for_red_hat_developer_hub/index)
  - **Specific section**: Search page for "mirror" or "air-gapped" to find air-gap configuration sections
  **Why This Needs Updates**:
  - **Current Issue**: States "The `prepare-restricted-environment.sh` script does not automatically parse Developer Lightspeed for RHDH images from the bundle manifest, so mirror these images manually"
  - **Root Cause**: LightSpeed images not in CSV `spec.relatedImages` (this ADR fixes that)
  - **Helm Issue**: References deprecated `global.lightspeed.`* field names instead of current `intelligentAssistant.initContainer.image`, `intelligentAssistant.sidecar.image`, `intelligentAssistant.okp.image`
  **Updates Needed**:
  - **Operator**: Remove manual bundle extraction requirement; update to reference CSV `spec.relatedImages` as source (once images are added per this ADR)
  - **Helm**: Replace `global.lightspeed.`* with current chart structure from `[values.yaml](https://github.com/redhat-developer/rhdh-chart/blob/main/charts/redhat-developer-hub/values.yaml)`
  - Document all 3 container images: `lightspeed-rag-init`, `lightspeed-core`, `okp`
  - Align with standard operator/Helm mirror flows (same manual-first approach as general air-gap docs)
4. **Orchestrator Air-Gap Documentation**:



  **Target Documentation Page**:

- **Verified URL**: [RHDH 1.10 - Install Orchestrator Plugin in Air-Gapped Environment with Operator](https://docs.redhat.com/en/documentation/red_hat_developer_hub/1.10/html/orchestrator_in_red_hat_developer_hub/install-orchestrator-plugin-in-an-air-gapped-environment-with-the-operator_orchestrator-in-rhdh)

  **Updates Needed**:

- **Clarify three-tier mirroring model**:
  - **Plugin OCI packages** → Manual `skopeo` commands or `[mirror-plugins.sh](https://github.com/redhat-developer/rhdh-operator/blob/main/.rhdh/scripts/mirror-plugins.sh)` (NOT in `relatedImages` - see [OLM container images requirement](https://docs.redhat.com/en/documentation/openshift_container_platform/4.4/html/operators/olm-restricted-networks))
  - **Serverless operator prerequisites** → Standard OLM catalog mirroring (they have their own CSV `relatedImages`)
  - **RHDH static container images** (if any) → Covered by RHDH CSV `relatedImages`
- Add explicit warning: *"Plugin packages are OCI artifacts (npm tarballs in OCI format), not container images. They cannot be listed in* `spec.relatedImages` *per [OLM specification](https://olm.operatorframework.io/docs/concepts/crds/clusterserviceversion/) and must be mirrored separately."*
- Document manual plugin mirroring workflow:
  ```bash
  # 1. Extract plugin catalog index
  skopeo copy oci://registry.access.redhat.com/rhdh/plugin-catalog-index:2.1.0 \
    oci://mirror.example.com/rhdh/plugin-catalog-index:2.1.0

  # 2. Extract plugin package references from index
  # 3. Mirror each orchestrator plugin package:
  skopeo copy \
    oci://registry.access.redhat.com/rhdh/red-hat-developer-hub-backstage-plugin-orchestrator@sha256:... \
    oci://mirror.example.com/rhdh/red-hat-developer-hub-backstage-plugin-orchestrator@sha256:...
  # (repeat for orchestrator-backend, scaffolder-backend-module-orchestrator, etc.)
  ```
- Link to [OpenShift Serverless documentation](https://docs.openshift.com/serverless/) for prerequisite operator installation
- Cross-reference `[mirror-plugins.sh](https://github.com/redhat-developer/rhdh-operator/blob/main/.rhdh/scripts/mirror-plugins.sh)` as optional automation with security review notice

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

