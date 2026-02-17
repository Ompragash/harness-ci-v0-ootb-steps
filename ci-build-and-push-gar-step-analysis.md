# Build and Push to GAR: Inputs & GCP Connector Mapping

This document traces the CI Manager implementation of the **Build and Push to GAR** step, its user-facing inputs, GCP connector auth flows, connector → plugin env mapping, and infra-type applicability.

All code locations below are workspace-relative `path:line` references.

type: BuildAndPushGAR
Plugin used: plugins/kaniko-gar:1.13.3 (K8), plugins/buildx-gar:1.4.2 (buildx), plugins/gar:21.1.1 (VM)

## Known Gaps / Gotchas

- The GAR connector provides auth, but the step still requires `host` and `projectID`, and the registry is constructed from those two step inputs.
- Registry construction: `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:941`
- In the GCP OIDC flow, `gcpProjectId` is mapped into `PLUGIN_PROJECT_NUMBER` (the env var name suggests “number,” but the source field is a project ID string).
- Mapping: `879-pipeline-ci-commons/src/main/java/io/harness/utils/GcpOidcAuthenticator.java:101`
- There is no separate “Cloud” infra type at this layer; hosted/DLITE runs through the VM path for env mapping.
- Infra types: `879-pipeline-ci-commons/src/main/java/io/harness/beans/sweepingoutputs/StageInfraDetails.java:33`
- VM/hosted uses `Type.VM`: `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/serializer/vm/VmPluginCompatibleStepSerializer.java:112`
- K8 uses `Type.K8`: `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/serializer/PluginCompatibleStepSerializer.java:76`
- Connector auth values are mapped to plugin-specific `PLUGIN_*` env vars (for example, `GCP_KEY` → `PLUGIN_JSON_KEY`), not standard GCP env names.
- GAR connector env var names: `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:712`

## Phase 1: Step Definition & Executor

Step registration and display metadata:
- `332-ci-manager/service/src/main/java/io/harness/ci/execution/plan/creator/CIPipelineServiceInfoProvider.java:376`

Step node wiring to the spec:
- `332-ci-manager/service/src/main/java/io/harness/ci/beans/steps/nodes/BuildAndPushGARNode.java:35`
- `332-ci-manager/service/src/main/java/io/harness/ci/beans/steps/nodes/BuildAndPushGARNode.java:44`

Plan creator:
- `332-ci-manager/service/src/main/java/io/harness/ci/execution/plancreator/BuildAndPushGARStepPlanCreator.java:19`

Executor (engine step):
- `332-ci-manager/service/src/main/java/io/harness/ci/execution/registrars/ExecutionRegistrar.java:93`
- `332-ci-manager/service/src/main/java/io/harness/ci/execution/states/GARStep.java:53`

User-facing spec model (inputs live here):
- `879-pipeline-ci-commons/src/main/java/io/harness/beans/steps/stepinfo/GARStepInfo.java:86`

Default retry value:
- `879-pipeline-ci-commons/src/main/java/io/harness/beans/steps/stepinfo/GARStepInfo.java:60`

Step env assembly entry point for GAR:
- `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:336`
- `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:933`

## Phase 2: GCP Connector Auth Methods & Required Fields

Supported GCP credential types:
- `954-connector-beans/src/main/java/io/harness/delegate/beans/connector/gcpconnector/GcpCredentialType.java:13`

Required fields by auth method:
- Manual credentials require `secretKeyRef`: `954-connector-beans/src/main/java/io/harness/delegate/beans/connector/gcpconnector/GcpManualDetailsDTO.java:31`
- OIDC requires `workloadPoolId`: `954-connector-beans/src/main/java/io/harness/delegate/beans/connector/gcpconnector/GcpOidcDetailsDTO.java:29`
- OIDC requires `providerId`: `954-connector-beans/src/main/java/io/harness/delegate/beans/connector/gcpconnector/GcpOidcDetailsDTO.java:30`
- OIDC requires `gcpProjectId`: `954-connector-beans/src/main/java/io/harness/delegate/beans/connector/gcpconnector/GcpOidcDetailsDTO.java:31`
- OIDC requires `serviceAccountEmail`: `954-connector-beans/src/main/java/io/harness/delegate/beans/connector/gcpconnector/GcpOidcDetailsDTO.java:32`
- Inherit from delegate requires delegate selectors on the spec: `954-connector-beans/src/main/java/io/harness/delegate/beans/connector/gcpconnector/GcpDelegateDetailsDTO.java:27`
- Inherit from delegate validation enforces selectors: `954-connector-beans/src/main/java/io/harness/delegate/beans/connector/GcpConnectorDTO.java:82`

Connector details assembly (auth-type switch):
- GCP connector details by auth type: `879-pipeline-ci-commons/src/main/java/io/harness/ci/utils/BaseConnectorUtils.java:645`

## Phase 3: Connector → Plugin Env Mapping (Per Auth Flow)

### Mapping Chain Overview

Connector env var names are decided by step type:
- GAR secret env name mapping: `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:712`

The mapping is applied on both K8 and VM paths:
- K8 conversion info uses the mapping: `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/integrationstage/k8s/K8InitializeStepUtils.java:2220`
- VM serializer merges the mapping into connector details: `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/serializer/vm/VmPluginCompatibleStepSerializer.java:267`

OIDC values are generated and injected through connector details:
- OIDC env map population: `879-pipeline-ci-commons/src/main/java/io/harness/utils/GcpOidcAuthenticator.java:101`
- OIDC map attached to connector details: `879-pipeline-ci-commons/src/main/java/io/harness/ci/utils/BaseConnectorUtils.java:658`
- OIDC envs are pulled into step envs: `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:2473`
- OIDC envs are merged into GAR step envs early: `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:936`

Manual credentials are materialized as secrets:
- Manual GCP secret mapping into env vars: `933-ci-commons/src/main/java/io/harness/connector/ConnectorEnvVariablesHelper.java:115`

### Auth Flow Details

Auth Method: Manual Credentials (Service Account Key)
- Connector Fields Used: `credential.spec.secretKeyRef`.
- Mapped to Plugin Env/Args: `PLUGIN_JSON_KEY`.
- Code Location: env var name mapping `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:716`; secret value population `933-ci-commons/src/main/java/io/harness/connector/ConnectorEnvVariablesHelper.java:133`.

Auth Method: OIDC / Workload Identity Federation
- Connector Fields Used: `workloadPoolId`, `providerId`, `gcpProjectId`, `serviceAccountEmail`.
- Mapped to Plugin Env/Args: `PLUGIN_PROJECT_NUMBER`, `PLUGIN_PROVIDER_ID`, `PLUGIN_POOL_ID`, `PLUGIN_SERVICE_ACCOUNT_EMAIL`, `PLUGIN_OIDC_TOKEN_ID`.
- Code Location: env map creation `879-pipeline-ci-commons/src/main/java/io/harness/utils/GcpOidcAuthenticator.java:101`; attached to connector details `879-pipeline-ci-commons/src/main/java/io/harness/ci/utils/BaseConnectorUtils.java:658`; injected into step envs `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:2473`.

Auth Method: IAM via Delegate (Inherit From Delegate)
- Connector Fields Used: `delegateSelectors`.
- Mapped to Plugin Env/Args: none directly (delegate identity is used).
- Code Location: spec selectors required `954-connector-beans/src/main/java/io/harness/delegate/beans/connector/gcpconnector/GcpDelegateDetailsDTO.java:27`; validation enforcement `954-connector-beans/src/main/java/io/harness/delegate/beans/connector/GcpConnectorDTO.java:82`.

## Phase 4: Plugin Inputs Summary (All Inputs)

Primary GAR env assembly:
- `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:933`

GAR-specific plugin settings:
- Registry construction: `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:945`
- Registry type marker: `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:992`

Plugin images configured for GAR:
- Kaniko GAR: `332-ci-manager/config/ci-manager-config.yml:392`
- Buildx GAR: `332-ci-manager/config/ci-manager-config.yml:421`

### Step Inputs → Plugin Env Vars

| Input Name | Type | Required | Description | Maps To (Plugin) |
|---|---|---:|---|---|
| connectorRef | string | Yes | GCP connector reference | Connector-derived envs (see auth flow matrix) |
| host | string | Yes | GAR host (for example, `us-docker.pkg.dev`) | Contributes to `PLUGIN_REGISTRY` (`PluginSettingUtils.java:941`, `PluginSettingUtils.java:948`) |
| projectID | string | Yes | GCP project ID | Contributes to `PLUGIN_REGISTRY` (`PluginSettingUtils.java:942`, `PluginSettingUtils.java:948`) |
| imageName | string | Yes | GAR repository/image name | `PLUGIN_REPO` (`PluginSettingUtils.java:950`) |
| tags | list<string> | Yes | Image tags | `PLUGIN_TAGS` (`PluginSettingUtils.java:953`) |
| dockerfile | string | No | Dockerfile path | `PLUGIN_DOCKERFILE` (`PluginSettingUtils.java:956`) |
| context | string | No | Build context | `PLUGIN_CONTEXT` (`PluginSettingUtils.java:962`) |
| target | string | No | Docker target stage | `PLUGIN_TARGET` (`PluginSettingUtils.java:967`) |
| buildArgs | map<string,string> | No | Docker build args | `PLUGIN_BUILD_ARGS`, `PLUGIN_BUILD_ARGS_NEW` (`PluginSettingUtils.java:972`) |
| labels | map<string,string> | No | Image labels | `PLUGIN_CUSTOM_LABELS` (`PluginSettingUtils.java:979`) |
| optimize | boolean | No | K8 snapshot optimization (defaults true) | `PLUGIN_SNAPSHOT_MODE=redo` on K8 (`PluginSettingUtils.java:1019`) |
| caching | boolean | No | Enables buildx/DLC behavior | Buildx driver opts + cache metrics (`PluginSettingUtils.java:1061`) |
| cacheFrom | list<string> | No | Cache sources | `PLUGIN_CACHE_FROM` (`PluginSettingUtils.java:1045`) |
| cacheTo | string | No | Cache destination | `PLUGIN_CACHE_TO` (`PluginSettingUtils.java:1054`) |
| remoteCacheImage | string | No | Remote cache image | `PLUGIN_ENABLE_CACHE` + `PLUGIN_CACHE_REPO` (K8 path) (`PluginSettingUtils.java:1026`) |
| baseImageConnectorRefs | list<string> | No | Base image connector(s) | Base image registry envs (`PluginSettingUtils.java:985`) |
| envDockerSecrets | map<string,string> | No | Docker secrets via env (buildx) | `PLUGIN_SECRETS_FROM_ENV` (`PluginSettingUtils.java:988`) |
| fileDockerSecrets | map<string,string> | No | Docker secrets via file (buildx) | `PLUGIN_SECRETS_FROM_FILE` (`PluginSettingUtils.java:988`) |
| envVariables | map<string,string> | No | Extra env vars | Merged directly (`PluginSettingUtils.java:994`) |
| runAsUser | integer | No | Container user override | Container runtime setting, not plugin env (`VmPluginCompatibleStepSerializer.java:112`) |
| resources | object | No | Container resources | Container runtime setting (`GARStepInfo.java:87`) |
| retry | integer | No | Step retry count | Defaults to 1 (`GARStepInfo.java:60`) |

### Auth Flow Matrix

| Auth Method | Connector Fields | Plugin Env Vars | Notes |
|---|---|---|---|
| Manual Credentials | `secretKeyRef` | `PLUGIN_JSON_KEY` | Env names: `PluginSettingUtils.java:716`; values: `ConnectorEnvVariablesHelper.java:133` |
| OIDC / Workload Identity | `workloadPoolId`, `providerId`, `gcpProjectId`, `serviceAccountEmail` | `PLUGIN_PROJECT_NUMBER`, `PLUGIN_PROVIDER_ID`, `PLUGIN_POOL_ID`, `PLUGIN_SERVICE_ACCOUNT_EMAIL`, `PLUGIN_OIDC_TOKEN_ID` | Created: `GcpOidcAuthenticator.java:101`; injected: `PluginSettingUtils.java:2473` |
| Inherit From Delegate | `delegateSelectors` | None directly | Selectors required: `GcpDelegateDetailsDTO.java:27`; enforced: `GcpConnectorDTO.java:82` |

## Infra Applicability (K8 vs VM vs Hosted/DLITE)

Infra types for CI steps are:
- `K8`, `VM`, and `DLITE_VM`: `879-pipeline-ci-commons/src/main/java/io/harness/beans/sweepingoutputs/StageInfraDetails.java:33`

K8 flows call the plugin env builder with `Type.K8`:
- `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/serializer/PluginCompatibleStepSerializer.java:76`

VM and hosted/DLITE flows call it with `Type.VM`:
- `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/serializer/vm/VmPluginCompatibleStepSerializer.java:112`

The GAR step’s env assembly explicitly branches on infra type:
- `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:999`

### Applicability Matrix

| Concern | K8 | VM | Hosted/DLITE_VM | Notes |
|---|---:|---:|---:|---|
| Connector auth env name mapping (`GCP_KEY` → `PLUGIN_JSON_KEY`) | Yes | Yes | Yes (via VM path) | Central mapping: `PluginSettingUtils.java:712`; K8 uses it: `K8InitializeStepUtils.java:2220`; VM merges it: `VmPluginCompatibleStepSerializer.java:267` |
| OIDC env injection (`PLUGIN_OIDC_TOKEN_ID`, `PLUGIN_*_ID` fields) | Yes | Yes | Yes (via VM path) | OIDC injected before step-specific envs: `PluginSettingUtils.java:936`, `PluginSettingUtils.java:2473` |
| Core GAR plugin envs (`PLUGIN_REGISTRY`, `PLUGIN_REPO`, `PLUGIN_TAGS`, etc.) | Yes | Yes | Yes (via VM path) | Common assembly in `getGARStepInfoEnvVariables`: `PluginSettingUtils.java:933` |
| K8-only envs/behaviors | Yes | No | No | K8 branch: `PluginSettingUtils.java:1001`; snapshot mode: `PluginSettingUtils.java:1019`; artifact file: `PluginSettingUtils.java:1015` |
| VM-only envs/behaviors | No | Yes | Yes (via VM path) | VM branch: `PluginSettingUtils.java:1003`; daemon off: `PluginSettingUtils.java:1038` |

## UI Field / Input

| Field / input | KubernetesDirect | KubernetesHosted | Cloud | VM | Docker | Optional? | Feature-flag impact | License impact | Tooltip ID | Tooltip content |
|---|---:|---:|---:|---:|---:|---|---|---|---|---|
| `identifier` | Y | Y | Y | Y | Y | No (Required) | None | None | None | None |
| `name` | Y | Y | Y | Y | Y | No (Required) | None | None | None | None |
| `spec.connectorRef` | Y | Y | Y | Y | Y | No (Required) | None | None | None *(Edit)* / `garConnector` *(Input Set)* | `garConnector` text not present in loaded tooltip dictionary |
| `spec.host` | Y | Y | Y | Y | Y | No (Required) | None | None | `gcrHost` | GCR/GAR hostname guidance |
| `spec.projectID` | Y | Y | Y | Y | Y | No (Required) | None | None | `gcrProjectID` | Google Cloud Project ID guidance |
| `spec.imageName` | Y | Y | Y | Y | Y | No (Required) | None | None | `imageName` | Image name with optional tag/digest format |
| `spec.tags` | Y | Y | Y | Y | Y | No (Required) | None | None | None *(Edit)* | None |
| `spec.caching` | Y | Y | Y | Y | Y | Yes | `CI_ENABLE_INTELLIGENT_DEFAULTS` can auto-set `true` for new steps when unset | None | None | None |
| `spec.baseImageConnectorRefs` | Y | Y | Y | Y | Y | Yes | None | None | `baseImageConnectorRefs` | Authenticated base-image pull connector guidance |
| `spec.optimize` | Y | Y | N | N | N | Yes | None | None | `optimize` | Enables Kaniko snapshot optimization |
| `spec.dockerfile` | Y | Y | Y | Y | Y | Yes | None | None | `dockerfile` | Dockerfile path/name guidance |
| `spec.context` | Y | Y | Y | Y | Y | Yes | None | None | `context` | Docker build context path guidance |
| `spec.labels` | Y | Y | Y | Y | Y | Yes | None | None | None | None |
| `spec.buildArgs` | Y | Y | Y | Y | Y | Yes | None | None | None | None |
| `spec.target` | Y | Y | Y | Y | Y | Yes | None | None | `target` | Docker target build stage |
| `spec.remoteCacheImage` | Y | Y | N | N | N | Yes | None | None | `gcrRemoteCache` | Remote Docker layer cache image format/constraints |
| `spec.envVariables` | Y | Y | Y | Y | Y | Yes | None | None | None | None |
| `spec.runAsUser` | Y | N | Y | N | N | Yes | None | None | `runAsUser` | Run-as user ID guidance |
| `spec.limitMemory` | Y | Y | N | N | N | Yes | None | None | `limitMemory` | Container memory limit format/default guidance |
| `spec.limitCPU` | Y | Y | N | N | N | Yes | None | None | `limitCPULabel` | Container CPU limit format/default guidance |
| `timeout` | Y | Y | Y | Y | Y | Yes | None | None | None *(Edit)* | None |

## Key Takeaways

- The connector auth mappings (manual credentials and OIDC) are applied on both K8 and VM/hosted flows.
- The step’s plugin env inputs are mostly shared, but there are infra-specific envs in the K8 and VM branches.
- For GAR specifically, host and projectID are required step inputs even when a connector is present.
