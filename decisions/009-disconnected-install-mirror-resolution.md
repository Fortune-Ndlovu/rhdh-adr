# ADR-009: Disconnected Install: Unified Mirror Resolution (Operator)

## Executive Summary / TL;DR

Extend RHDH Operator's existing ConfigMap-based mirror resolution (currently used for dynamic plugin packages) to include container images for flavours and optional components. Address Orchestrator documentation gap for vanilla Kubernetes air-gapped deployments. This provides a unified, dynamic approach to air-gapped deployments without requiring CSV maintenance or forcing users to mirror unused optional features.

**Current State**: The operator already implements mirror resolution via the `plugin-registry-mirror` ConfigMap for plugin OCI packages ([implementation](https://github.com/redhat-developer/rhdh-operator/blob/main/pkg/model/dynamic-plugins-mirror.go)). Users create a ConfigMap with mirror mappings, and the operator automatically rewrites plugin package URLs during reconciliation.

**Proposed Extension**: Apply the same ConfigMap-based resolution to container images (Lightspeed Core sidecar, OKP, PostgreSQL) by adding mirror substitution in each object's `addToModel()` method.

**Why Not spec.relatedImages**: While OpenShift recommends listing images in CSV `spec.relatedImages`, this approach has limitations for RHDH:
- Forces mirroring of large optional images (OKP is 10+ GB) even when unused
- Requires CSV updates whenever image versions change
- Doesn't work on platforms where IDMS/ITMS are blocked (e.g., Rosa HCP)
- Creates maintenance burden for flavour combinations

**Key Decisions**:
1. **Runtime mirror resolution** at reconciliation time, using the same ConfigMap infrastructure already in place for plugins
2. **Manual-first documentation** with transparent bash snippets; scripts (`prepare-restricted-environment.sh`, `mirror-plugins.sh`) positioned as optional automation

---

## Context

### Problem Statement

RHDH customers on disconnected or restricted networks face two gaps:

**Container images** (Intelligent Assistant):
1. Images defined in flavour/plugin-deps ConfigMaps, not discoverable from CSV alone
2. No runtime mirror resolution for container images (only plugin packages have this)
3. Current workaround requires manual bundle extraction and `RELATED_IMAGE_*` env var overrides

**Orchestrator on Kubernetes**:
1. Plugin OCI packages already support mirror resolution (via existing ConfigMap mechanism)
2. Documentation gap: [Air-gap instructions](https://docs.redhat.com/en/documentation/red_hat_developer_hub/1.10/html/orchestrator_in_red_hat_developer_hub/install-orchestrator-plugin-in-an-air-gapped-environment-with-the-operator_orchestrator-in-rhdh) only cover OpenShift with oc-mirror
3. Missing: Vanilla Kubernetes air-gap procedure using skopeo + Helm chart

### Evidence of Current Gap

From [RHDH 1.10 LightSpeed air-gap documentation](https://docs.redhat.com/en/documentation/red_hat_developer_hub/1.10/html/interacting_with_red_hat_developer_lightspeed_for_red_hat_developer_hub/configure-to-initialize-the-ai-assistant_interacting-with-developer-lightspeed-for-rhdh#mirror-images-for-air-gapped-environments_configure-to-initialize-the-ai-assistant):

> "The `prepare-restricted-environment.sh` script does **not automatically parse Developer Lightspeed for RHDH images from the bundle manifest**, so mirror these images manually before running the script."

### Current Mirror Resolution (Plugins)

The operator already implements ConfigMap-based mirror resolution:

**ConfigMap Format** (IDMS/ITMS-compatible):
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: plugin-registry-mirror
  namespace: rhdh-operator
data:
  mirrors.yaml: |
    imageMirrors:
      - source: registry.access.redhat.com/rhdh
        mirrors:
          - mirror.example.com/rhdh-plugins
```

**How It Works**:
1. ConfigMap mounted into operator pod at `/plugins-mirror/mirrors.yaml`
2. Loaded during reconciliation preprocessing ([spec_preprocessor.go:157](https://github.com/redhat-developer/rhdh-operator/blob/main/internal/controller/spec_preprocessor.go#L157))
3. Stored in `model.ExternalConfig.Mirrors`
4. Applied to plugin packages during dynamic plugin processing ([dynamic-plugins.go:176](https://github.com/redhat-developer/rhdh-operator/blob/main/pkg/model/dynamic-plugins.go#L176))

**Current Scope**: Only `oci://` plugin package references  
**Proposed Extension**: Add container image references (without `oci://` prefix)

### Container Images Requiring Mirror Support

**Intelligent Assistant / Lightspeed**:
- `quay.io/lightspeed-core/lightspeed-stack:0.7.0` (sidecar)
- `registry.redhat.io/offline-knowledge-portal/rhokp-rhel9:1.2.12` (10+ GB, optional)
- `registry.access.redhat.com/rhdh/rhdh-lightspeed-rag-content-init-rhel9@sha256:...` (init container)

**Core Components** (already have `RELATED_IMAGE_*` workaround):
- `quay.io/rhdh-community/rhdh:next` (main backstage container)
- `quay.io/fedora/postgresql-18:latest` (if local DB enabled)

**Orchestrator**:
- No container images - uses plugin OCI packages only
- Plugin packages already covered by existing mirror resolution
- **Gap**: Missing vanilla Kubernetes air-gap documentation (OpenShift-only currently documented)

---

## Decision

**Apply runtime mirror resolution to container images during reconciliation**, extending the existing ConfigMap-based approach already used for plugin packages.

Users create one `plugin-registry-mirror` ConfigMap with registry mappings. The operator reads it once at startup, then applies mirror substitution to all container image references (containers, init containers, sidecars) when building Kubernetes objects for each Backstage CR. Pods receive already-resolved registry paths—no IDMS/ITMS required.

This keeps mirroring dynamic (ConfigMap changes apply on next reconciliation), optional (only mirror what you enable), and platform-agnostic (works everywhere the operator runs).

### Architectural Approach

**When**: During reconciliation, after flavour manifests are merged but before objects are applied to Kubernetes  
**Where**: In each object's initialization phase (Deployment, StatefulSet, etc.)  
**How**: Read mirror mappings from `model.ExternalConfig.Mirrors`, apply string substitution to container image fields

**Key principle**: Mirror resolution happens at the operator level, not the pod level. The operator rewrites image references before creating Kubernetes objects, so pods see already-resolved registry paths.

**Scope**:
- Container images in Deployments (backstage, lightspeed-core)
- Container images in StatefulSets (postgresql)
- Container images in plugin-deps (OKP)
- Init container images
- Documentation for Orchestrator air-gap on vanilla Kubernetes

**Out of scope** (already handled):
- Plugin OCI packages mirror resolution (existing implementation covers Orchestrator plugins)
- npm packages, HTTP URLs (not container images)

### Implementation Strategy

Apply mirror resolution during object initialization, following the pattern already used for plugins:

1. **Extend mirror resolution**: Modify existing mirror logic to handle both `oci://` prefixed (plugins) and plain (containers) image references
2. **Add to Deployment**: Apply mirrors to all containers and init containers during `addToModel()`
3. **Add to StatefulSet**: Apply mirrors to PostgreSQL container during `addToModel()`
4. **Verify plugin-deps**: Confirm OKP deployment flows through same reconciliation path

**Orchestrator-specific deliverables**:

Orchestrator has no container images to mirror—only plugin OCI packages, already covered by existing mirror resolution. However, documentation gaps remain:

- **Current state**: [Orchestrator air-gap docs](https://docs.redhat.com/en/documentation/red_hat_developer_hub/1.10/html/orchestrator_in_red_hat_developer_hub/install-orchestrator-plugin-in-an-air-gapped-environment-with-the-operator_orchestrator-in-rhdh) cover OpenShift with oc-mirror only
- **Required**: Add vanilla Kubernetes air-gap procedure:
  1. Discover plugin packages from bundle (extract `dynamic-plugins.yaml` from flavour-orchestrator)
  2. Mirror OCI packages with `skopeo copy`
  3. Create `plugin-registry-mirror` ConfigMap (same as Lightspeed)
  4. Deploy on Kubernetes with Helm chart
  5. Verify: Orchestrator plugins load from mirror, workflows execute offline

**Acceptance test**: Deploy RHDH with Orchestrator on vanilla Kubernetes (kind/minikube), no internet access, using only mirrored artifacts.

**Documentation approach**:

All air-gap documentation follows a **manual-first** pattern:

1. **Prerequisites**: List required tools (`yq`, `skopeo`/`oc-mirror`, `kubectl`/`oc`)
2. **Discovery**: Show clear `yq` commands to extract image lists from bundle
3. **Mirroring**: Show transparent `skopeo copy` or `oc-mirror` commands (10-20 lines max)
4. **Resolution**: Show ConfigMap creation (simple YAML example)
5. **Deployment**: Show Backstage CR creation
6. **Verification**: Show how to confirm images pulled from mirror
7. **Optional automation**: Link to `prepare-restricted-environment.sh` or `mirror-plugins.sh` with note: "For automation, see [script-name]"

**Rationale**: Enterprise customers with security policies against running external scripts need auditable, controllable steps. Scripts become helpers, not requirements.

**Current docs problem**: Today's [air-gap operator docs](https://docs.redhat.com/en/documentation/red_hat_developer_hub/1.10/html-single/installing_red_hat_developer_hub_in_an_air-gapped_environment/index#install-rhdh-in-an-air-gapped-environment-with-the-operator_installing-rhdh-in-an-air-gapped-environment) ONLY show script usage. This ADR requires complete rewrite to manual-first approach.

Detailed implementation in companion guide: `/home/fndlovu/Documents/rhdh-airgap-strategy/mirror-resolution-complete-guide.md`

### User Workflow

#### Discovery: Finding Required Images

Users discover images by inspecting the operator bundle manifests. The bundle contains all flavour and plugin-deps configurations.

**Extract bundle**:
```bash
oc image extract registry.redhat.io/rhdh/rhdh-operator-bundle:v1.4.0 \
  --path /manifests/:./bundle
```

**Find container images**:
```bash
# Core images (always present)
yq eval '.data."deployment.yaml"' bundle/manifests/backstage-operator.clusterserviceversion.yaml | \
  grep "image:" | grep -v "imagePullPolicy"

yq eval '.data."db-statefulset.yaml"' bundle/manifests/backstage-operator.clusterserviceversion.yaml | \
  grep "image:" | grep -v "imagePullPolicy"

# Flavour images (intelligent-assistant, orchestrator, etc.)
yq eval '.data."flavour-intelligent-assistant"' bundle/manifests/backstage-operator.clusterserviceversion.yaml | \
  grep "image:" | grep -v "imagePullPolicy"

# Plugin dependency images (OKP, etc.)
yq eval '.data."plugin-deps-okp"' bundle/manifests/backstage-operator.clusterserviceversion.yaml | \
  grep "image:" | grep -v "imagePullPolicy"
```

**Note**: Only mirror images for flavours/features you plan to enable. For example, skip OKP (10+ GB) if not using intelligent-assistant with offline knowledge portal.

#### OpenShift: Mirroring

**Mirror images**:
```bash
# Using oc-mirror
cat <<EOF > imageset-config.yaml
apiVersion: mirror.openshift.io/v2alpha1
kind: ImageSetConfiguration
mirror:
  additionalImages:
    - name: quay.io/lightspeed-core/lightspeed-stack:0.7.0
    - name: registry.redhat.io/offline-knowledge-portal/rhokp-rhel9:1.2.12
EOF

oc-mirror --config imageset-config.yaml docker://mirror.company.com
```

**Create mirror ConfigMap** (one-time setup):
```bash
cat <<EOF | oc apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: plugin-registry-mirror
  namespace: rhdh-operator
data:
  mirrors.yaml: |
    imageMirrors:
      - source: quay.io/lightspeed-core
        mirrors:
          - mirror.company.com/lightspeed-core
      
      - source: registry.redhat.io/offline-knowledge-portal
        mirrors:
          - mirror.company.com/okp
      
      - source: registry.access.redhat.com/rhdh
        mirrors:
          - mirror.company.com/rhdh-plugins
EOF

# Restart operator to mount the ConfigMap
oc rollout restart deployment rhdh-operator -n rhdh-operator
oc rollout status deployment rhdh-operator -n rhdh-operator
```

**Deploy Backstage** - mirrors applied automatically on reconciliation

#### Vanilla Kubernetes

Same workflow, but use `skopeo copy` for mirroring:

**Container images**:
```bash
skopeo copy \
  docker://quay.io/lightspeed-core/lightspeed-stack:0.7.0 \
  docker://mirror.company.com/lightspeed-core/lightspeed-stack:0.7.0
```

**Plugin OCI packages** (Orchestrator):
```bash
# Extract plugin list from bundle
yq eval '.data."flavour-orchestrator"' bundle/manifests/backstage-operator.clusterserviceversion.yaml \
  > orchestrator-plugins.yaml

# Find OCI package references
grep "oci://" orchestrator-plugins.yaml

# Mirror each plugin package
skopeo copy \
  oci://registry.access.redhat.com/rhdh/red-hat-developer-hub-backstage-plugin-orchestrator@sha256:... \
  oci://mirror.company.com/rhdh/red-hat-developer-hub-backstage-plugin-orchestrator@sha256:...
```

**Same ConfigMap works for both**:
```yaml
imageMirrors:
  - source: registry.access.redhat.com/rhdh
    mirrors:
      - mirror.company.com/rhdh
```

---

## Alternatives Considered

### Alternative 1: Extend spec.relatedImages in CSV

**Approach**: List all container images in the operator's CSV under `spec.relatedImages` and rely on OpenShift's IDMS/ITMS for resolution.

**Advantages**:
- Follows OpenShift's recommended pattern for disconnected operators
- OLM automatically handles image mirroring during operator install
- Clear discovery path via CSV

**Rejected because**:
1. **Forces mirroring of unused features**: OKP image is 10+ GB. Listing it in CSV means every user must mirror it, even those who never enable intelligent-assistant flavour.

2. **CSV maintenance burden**: Every flavour image version change requires CSV update and bundle rebuild.

3. **Platform limitations**: IDMS/ITMS don't work on Rosa HCP (hosted control planes). ConfigMap approach works everywhere.

4. **Inconsistent with plugin approach**: Plugins already use ConfigMap-based resolution. Having two different approaches (CSV for containers, ConfigMap for plugins) creates user confusion.

5. **Not dynamic**: Users must upgrade operator to get new mirror configurations. ConfigMap updates take effect immediately.

**Team feedback** ([PR review](https://github.com/redhat-developer/rhdh-adr/pull/29#pullrequestreview-5529579002)):
> "Mirror configuration should be a first class citizen for RHDH Operator. The best approach is configure mirrors on operator level (using ConfigMap) and make operator apply it dynamically. It is already applied for dynamic plugins and we can extend it for others."

### Alternative 2: RELATED_IMAGE_* Environment Variables Only

**Approach**: Extend the existing `RELATED_IMAGE_backstage` pattern to all images (e.g., `RELATED_IMAGE_lightspeed_core`, `RELATED_IMAGE_okp`).

**Advantages**:
- Already implemented pattern
- Subscription/ClusterExtension override mechanism exists

**Rejected because**:
1. **Not dynamic**: Requires operator pod restart to change
2. **Verbose**: Each image needs separate env var
3. **Manual**: Users must set env vars for every flavour image
4. **Doesn't scale**: Flavour combinations create exponential env var requirements

### Alternative 3: Update prepare-restricted-environment.sh to Read spec.relatedImages

**Approach**: Add Lightspeed/OKP images to CSV `spec.relatedImages`, update `prepare-restricted-environment.sh` to read from CSV first, fallback to ConfigMap parsing.

**Advantages**:
- Centralized image list in CSV
- Script automation for mirroring

**Rejected because**:
1. **Doesn't solve resolution**: Script mirrors images, but operator still needs ConfigMap for runtime resolution (IDMS/ITMS don't work everywhere)
2. **Forces OKP mirroring**: 10+ GB image must be mirrored even if user doesn't enable intelligent-assistant
3. **Duplicate maintenance**: Images in both CSV and ConfigMaps; downstream checks both locations
4. **Complexity**: Requires CSV changes, script changes, AND operator changes (for resolution)

**ConfigMap-only approach is simpler**: Images stay in ConfigMaps (already there), operator reads ConfigMap for resolution (new), no CSV changes needed.

### Alternative 4: Document Manual Steps Only

**Approach**: Document how to extract images from bundles and manually patch deployments.

**Rejected because**:
- Requires per-deployment manual intervention
- Error-prone
- Doesn't scale
- Contradicts operator's purpose (automation)

---

## Consequences

### Positive

✅ **Single ConfigMap for all mirroring**: Users create one ConfigMap for plugins AND container images  
✅ **Optional feature support**: Only mirror images for features actually enabled  
✅ **Platform-agnostic**: Works on OpenShift (including Rosa HCP), vanilla Kubernetes, anywhere operator runs  
✅ **Consistent approach**: Same mechanism for plugins and containers  
✅ **IDMS/ITMS compatible format**: Users familiar with OpenShift disconnected environments recognize the format  
✅ **Backward compatible**: Existing plugin mirroring continues to work; new container mirroring is additive  
✅ **No CSV maintenance**: Image version changes don't require operator bundle updates  
✅ **No operator upgrade needed**: ConfigMap changes apply on next Backstage CR reconciliation (operator restart only required for initial ConfigMap creation)  
✅ **Orchestrator coverage**: Plugin packages already supported; adds missing K8s documentation to match OpenShift parity  
✅ **Transparent documentation**: Manual-first approach gives users auditable, controllable steps; scripts become optional helpers  

### Negative

❌ **Not discoverable via CSV**: Images not listed in `spec.relatedImages` (but discoverable via bundle manifest inspection)  
❌ **Requires operator code changes**: Need to implement mirror application in deployment/statefulset `addToModel()` methods  
❌ **Documentation updates**: Need to document ConfigMap-based approach for both discovery and resolution  
❌ **Testing complexity**: Need integration tests for various flavour combinations  

### Neutral

⚖️ **Discovery from bundle manifests**: Images not in CSV, but discoverable via bundle inspection (requires `yq` or similar tooling)  
⚖️ **Initial setup requires operator restart**: ConfigMap must exist before operator starts to be mounted; updates to ConfigMap apply on next Backstage reconciliation  
⚖️ **Existing `RELATED_IMAGE_*` env vars remain**: Still work for users who prefer env var override approach  

---

## Implementation Repositories

### Operator Changes (rhdh-operator)

| Concern | Files | What Changes | Validation |
|---------|-------|--------------|------------|
| **Mirror resolution logic** | `pkg/model/dynamic-plugins-mirror.go` | Extend `ApplyMirror()` to handle both `oci://` (plugins) and plain (containers) image refs | Unit tests: container image refs with tags/digests |
| **Deployment mirror application** | `pkg/model/deployment.go:82` | Add `applyMirrorsToAllContainers()` call in `addToModel()` after existing `RELATED_IMAGE_backstage` override | Verify Lightspeed sidecar uses mirrored registry |
| **StatefulSet mirror application** | `pkg/model/db-statefulset.go` | Add `applyMirrorsToAllContainers()` call in `addToModel()` after existing `RELATED_IMAGE_postgresql` override | Verify PostgreSQL uses mirrored registry |
| **Plugin-deps handling** | `internal/controller/plugin_dependencies.go` | Verify OKP deployment flows through same reconciliation path and receives mirror substitution | OKP deployment uses mirrored registry |
| **Unit tests** | `pkg/model/dynamic-plugins-mirror_test.go` | Add test cases for container image mirror resolution (with/without `oci://` prefix) | Coverage for boundary cases (ports, substrings, etc.) |
| **Integration tests** | `tests/e2e/` | Add air-gap scenario: Deploy Backstage with intelligent-assistant on vanilla K8s, verify all images pulled from mirror | E2E test passes with no internet access |
| **Operator docs (in-repo)** | `.rhdh/docs/airgap.adoc` | Update with ConfigMap approach (remove references to spec.relatedImages for flavours) | Doc review |

### Helm Chart (rhdh-chart)

| Concern | Files | What Changes | Validation |
|---------|-------|--------------|------------|
| **Chart values** | `charts/redhat-developer-hub/values.yaml` | **No changes needed** - Chart values already document all images | Existing `helm show values` workflow continues |
| **Chart README** | `charts/redhat-developer-hub/README.md` | **No changes needed** - Already documents air-gap via values.yaml | Doc review |

**Note**: Helm users rely on `values.yaml` as image inventory source (already documented). ConfigMap mirror resolution is operator-only.

### Documentation (red-hat-developers-documentation-rhdh)

**Documentation Philosophy**: Manual-first approach with scripts as optional automation.

**Rationale**: Enterprise customers often have security policies against running large external scripts (like `prepare-restricted-environment.sh`). They prefer transparent, auditable commands they control. Documentation should show clear manual steps first, then link to scripts as optional automation.

| Document | What Changes | Example Manual Steps |
|----------|--------------|---------------------|
| **Install RHDH in air-gap (Operator)** | **Manual-first**: Show step-by-step `yq`/`jq` to extract images from bundle, `oc-mirror`/`skopeo` to mirror, ConfigMap creation. **Then**: Link to `prepare-restricted-environment.sh` as optional | Extract: `yq eval '.data."deployment.yaml"'`<br>Mirror: `skopeo copy docker://... docker://...`<br>Create: `kubectl apply -f plugin-registry-mirror.yaml` |
| **Install RHDH + Intelligent Assistant (Operator)** | **Manual-first**: Show extracting Lightspeed images from flavour bundle, mirroring, ConfigMap. **Then**: Optional script. Remove old "manual extraction required" note | Extract: `yq eval '.data."flavour-intelligent-assistant"'`<br>Mirror: Same as above<br>ConfigMap: Same ConfigMap covers plugins + containers |
| **Install Orchestrator air-gap (Operator)** | Clarify that plugin packages are already handled by ConfigMap; show manual plugin extraction/mirroring. **Then**: Optional `mirror-plugins.sh` | Extract: `yq eval '.data."flavour-orchestrator"'`<br>Mirror: `skopeo copy oci://... oci://...` |
| **Install Orchestrator air-gap (Helm on K8s)** | **NEW DOCUMENT** - Manual vanilla K8s procedure: extract plugins, mirror with skopeo, create ConfigMap, deploy with Helm. **Then**: Optional `mirror-plugins.sh` | Same manual steps as Operator, but deploy via `helm install` |
| **Operator air-gap reference** | Add ConfigMap format reference; show manual vs automatic approaches side-by-side | ConfigMap YAML example with annotations |

**Key principle**: Each doc follows pattern:
1. **Prerequisites** (tools: yq, skopeo/oc-mirror, kubectl)
2. **Manual steps** (10-20 line bash snippets, clear and auditable)
3. **Verification** (how to confirm it worked)
4. **Optional automation** (link to `prepare-restricted-environment.sh` or `mirror-plugins.sh`)

**Current docs issue**: Today's docs ONLY show `prepare-restricted-environment.sh` script (see [Install RHDH air-gap](https://docs.redhat.com/en/documentation/red_hat_developer_hub/1.10/html-single/installing_red_hat_developer_hub_in_an_air-gapped_environment/index#install-rhdh-in-an-air-gapped-environment-with-the-operator_installing-rhdh-in-an-air-gapped-environment)). This ADR requires flipping that: manual steps first, script second.

### Scripts / Tooling (rhdh-operator)

| Script | What Changes | Why |
|--------|--------------|-----|
| `prepare-restricted-environment.sh` | **No changes needed** - Script already handles mirroring; ConfigMap is separate user step | Scripts mirror images; users create ConfigMap separately |
| `mirror-plugins.sh` | **No changes needed** - Already handles plugin OCI packages | No change to plugin mirroring |

**Note**: Scripts handle **mirroring** (copying images to mirror registry). ConfigMap handles **resolution** (operator rewriting image refs). These are separate concerns.

### Summary of Changes by Repository

| Repository | Operator Code | Bundle/Manifests | Scripts | Documentation |
|------------|---------------|------------------|---------|---------------|
| **rhdh-operator** | ✏️ **Changes**: Mirror application in deployment/statefulset | ✅ **No change**: Images stay in ConfigMaps | ✅ **No change**: Scripts continue to mirror | ✏️ **Major changes**: Manual-first approach + ConfigMap |
| **rhdh-chart** | N/A | ✅ **No change**: values.yaml already documents images | N/A | ✏️ **Changes**: Add Orchestrator K8s manual steps |
| **rhdh-docs** | N/A | N/A | N/A | ✏️ **Major changes**: Flip from script-first to manual-first across all air-gap docs |

**Key insights**: 
- Most changes are **documentation** and **operator runtime behavior**
- Bundle structure and mirroring scripts remain unchanged
- **Documentation philosophy shift**: From "run this script" to "here are the steps (script optional)"

---

## Downstream Impact

This ADR has **minimal downstream impact** because it does not modify the operator bundle's CSV or ConfigMap structure—only the operator's runtime behavior.

### Background: Downstream Image Handling

The downstream build process (midstream → product) performs two key operations:

1. **Digest pinning**: The `replaceUpstreamImages.sh` script converts upstream tags (`:latest`, `:0.7.0`) to digest-pinned product images (`@sha256:abc123...`)
2. **Staleness detection**: The `sync-midstream.sh` script compares image digests in bundle ConfigMaps against Red Hat's product registry, triggering rebuilds when images are outdated

### Current State

Container images for Lightspeed and OKP exist **only in bundle ConfigMaps** (flavour-intelligent-assistant, plugin-deps-okp), not in CSV `spec.relatedImages`. Downstream already:
- ✅ Digest-pins these ConfigMap images via `replaceUpstreamImages.sh`
- ✅ Checks ConfigMap images for staleness via `sync-midstream.sh`

### What Changes with This ADR

**Nothing in the bundle**: Images remain only in ConfigMaps, not added to CSV `spec.relatedImages`

**Runtime behavior only**: Operator code changes to read `plugin-registry-mirror` ConfigMap and apply mirrors during reconciliation. This ConfigMap is **user-created** (not shipped in bundle), so it doesn't affect downstream build.

### Required Downstream Changes

**None**. Downstream scripts already handle ConfigMap images correctly:
- ✅ `replaceUpstreamImages.sh` already digest-pins ConfigMap images
- ✅ `sync-midstream.sh` already checks ConfigMap images for staleness  
- ✅ No new images being added to bundle (Lightspeed/OKP already in ConfigMaps)

### Validation

Downstream build continues to:
1. Extract bundle ConfigMaps containing image references
2. Pin all image tags to digests
3. Check digest freshness
4. Rebuild when product images update

The `plugin-registry-mirror` ConfigMap (used for runtime resolution) is not part of the bundle and does not participate in downstream build process.

---

## References

### Operator Source Code (Current Implementation)

- [Mirror resolution logic](https://github.com/redhat-developer/rhdh-operator/blob/main/pkg/model/dynamic-plugins-mirror.go) - Current plugin mirror implementation
- [Plugin mirror application](https://github.com/redhat-developer/rhdh-operator/blob/main/pkg/model/dynamic-plugins.go#L176) - Where mirrors are applied to plugins
- [ConfigMap mount](https://github.com/redhat-developer/rhdh-operator/blob/main/config/manager/deployment.yaml#L103-L122) - How ConfigMap is mounted to operator
- [Mirror loading](https://github.com/redhat-developer/rhdh-operator/blob/main/internal/controller/spec_preprocessor.go#L157) - Where mirrors are loaded during reconciliation
- [Deployment model](https://github.com/redhat-developer/rhdh-operator/blob/main/pkg/model/deployment.go#L82) - Target for container image mirror application

### OpenShift Documentation

- [Disconnected Environments](https://docs.redhat.com/en/documentation/openshift_container_platform/4.17/html-single/disconnected_environments/index)
- [OC Mirror Plugin](https://docs.redhat.com/en/documentation/openshift_container_platform/4.17/html-single/disconnected_environments/index#installation-oc-mirror-installing-plugin_about-installing-oc-mirror-v2)

### RHDH Product Documentation

- [Air-Gapped Operator Installation](https://docs.redhat.com/en/documentation/red_hat_developer_hub/1.10/html-single/installing_red_hat_developer_hub_in_an_air-gapped_environment/index#install-rhdh-in-an-air-gapped-environment-with-the-operator_installing-rhdh-in-an-air-gapped-environment)
- [LightSpeed Air-Gap Configuration](https://docs.redhat.com/en/documentation/red_hat_developer_hub/1.10/html/interacting_with_red_hat_developer_lightspeed_for_red_hat_developer_hub/configure-to-initialize-the-ai-assistant_interacting-with-developer-lightspeed-for-rhdh#mirror-images-for-air-gapped-environments_configure-to-initialize-the-ai-assistant)
- [Orchestrator Air-Gap Installation](https://docs.redhat.com/en/documentation/red_hat_developer_hub/1.10/html/orchestrator_in_red_hat_developer_hub/install-orchestrator-plugin-in-an-air-gapped-environment-with-the-operator_orchestrator-in-rhdh)
---