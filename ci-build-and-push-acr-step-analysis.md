# Build and Push to ACR: Inputs & Azure Connector Mapping

This document traces the CI Manager implementation of the **Build and Push to ACR** step, its user-facing inputs, Azure connector auth flows, connector → plugin env mapping, and infra-type applicability.

All code locations below are workspace-relative `path:line` references.

type: BuildAndPushACR
Plugin used: plugins/kaniko-acr:1.13.3 (K8), plugins/buildx-acr:1.4.2 (buildx), plugins/acr:21.1.1 (VM)

## Known Gaps / Gotchas

- The ACR registry is derived from the `repository` string. `PLUGIN_REGISTRY` is the substring before `/`, and `PLUGIN_REPO` is the full repository string.
- `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:1085`
- The optional `subscriptionId` only affects the published artifact URL; it is not used for auth.
- `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:1086`
- `332-ci-manager/service/src/main/java/io/harness/ci/execution/states/ACRStep.java:119`
- OIDC env vars use `AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, `AZURE_AUDIENCE`, and `PLUGIN_OIDC_TOKEN_ID`, while manual auth uses `CLIENT_ID`, `TENANT_ID`, `CLIENT_SECRET`, or `CLIENT_CERTIFICATE`.
- OIDC map creation: `879-pipeline-ci-commons/src/main/java/io/harness/utils/AzureOidcAuthenticator.java:69`
- Manual env name mapping: `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:705`
- The ACR step reuses the GAR buildkit image constant when `caching=true` on VM.
- `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:1142`
- There is no separate “Cloud” infra type; hosted/DLITE flows go through the VM serializer path.
- Infra types: `879-pipeline-ci-commons/src/main/java/io/harness/beans/sweepingoutputs/StageInfraDetails.java:33`
- VM serializer path: `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/serializer/vm/VmPluginCompatibleStepSerializer.java:212`

## Phase 1: Step Definition & Executor

Step registration and display metadata:
- `332-ci-manager/service/src/main/java/io/harness/ci/execution/plan/creator/CIPipelineServiceInfoProvider.java:345`

Step node wiring to the spec:
- `332-ci-manager/service/src/main/java/io/harness/ci/beans/steps/nodes/BuildAndPushACRNode.java:35`
- `332-ci-manager/service/src/main/java/io/harness/ci/beans/steps/nodes/BuildAndPushACRNode.java:44`

Plan creator:
- `332-ci-manager/service/src/main/java/io/harness/ci/execution/plancreator/BuildAndPushACRStepPlanCreator.java:19`

Executor (engine step):
- `332-ci-manager/service/src/main/java/io/harness/ci/execution/registrars/ExecutionRegistrar.java:94`
- `332-ci-manager/service/src/main/java/io/harness/ci/execution/states/ACRStep.java:47`

User-facing spec model (inputs live here):
- `879-pipeline-ci-commons/src/main/java/io/harness/beans/steps/stepinfo/ACRStepInfo.java:58`

Default retry value:
- `879-pipeline-ci-commons/src/main/java/io/harness/beans/steps/stepinfo/ACRStepInfo.java:59`

Step env assembly entry point for ACR:
- `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:325`
- `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:1077`

## Phase 2: Azure Connector Auth Methods & Required Fields

Supported Azure credential types:
- `954-connector-beans/src/main/java/io/harness/delegate/beans/connector/azureconnector/constants/AzureCredentialType.java:17`

Required fields by auth method:

Manual credentials:
- Required fields: `clientId`, `tenantId`, and `auth`.
- `954-connector-beans/src/main/java/io/harness/delegate/beans/connector/azureconnector/AzureManualDetailsDTO.java:33`
- Secret key auth requires `secretRef`.
- `954-connector-beans/src/main/java/io/harness/delegate/beans/connector/azureconnector/AzureClientSecretKeyDTO.java:35`
- Certificate auth requires `certificateRef`.
- `954-connector-beans/src/main/java/io/harness/delegate/beans/connector/azureconnector/AzureClientKeyCertDTO.java:35`
- Supported secret types are `Secret` and `Certificate`.
- `954-connector-beans/src/main/java/io/harness/delegate/beans/connector/azureconnector/constants/AzureSecretType.java:18`

OIDC:
- Required fields: `tenantId`, `clientId`.
- `954-connector-beans/src/main/java/io/harness/delegate/beans/connector/azureconnector/AzureOidcSpecDTO.java:33`
- `audience` is optional with a default.
- `954-connector-beans/src/main/java/io/harness/delegate/beans/connector/azureconnector/AzureOidcSpecDTO.java:37`

Inherit from delegate (Managed Identity):
- Required field: `auth`.
- `954-connector-beans/src/main/java/io/harness/delegate/beans/connector/azureconnector/AzureInheritFromDelegateDetailsDTO.java:36`
- User-assigned MSI requires `clientId`.
- `954-connector-beans/src/main/java/io/harness/delegate/beans/connector/azureconnector/AzureUserAssignedMSIAuthDTO.java:31`

Connector detail assembly by auth type:
- `879-pipeline-ci-commons/src/main/java/io/harness/ci/utils/BaseConnectorUtils.java:599`

## Phase 3: Connector → Plugin Env Mapping (Per Auth Flow)

### Mapping Chain Overview

Connector env var names are decided by step type:
- ACR secret env name mapping: `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:705`

The mapping is applied on both K8 and VM paths:
- K8 conversion info uses the mapping: `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/integrationstage/k8s/K8InitializeStepUtils.java:2220`
- VM serializer merges the mapping into connector details: `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/serializer/vm/VmPluginCompatibleStepSerializer.java:267`

OIDC values are generated and injected through connector details:
- OIDC env map creation: `879-pipeline-ci-commons/src/main/java/io/harness/utils/AzureOidcAuthenticator.java:69`
- OIDC map attached to connector details: `879-pipeline-ci-commons/src/main/java/io/harness/ci/utils/BaseConnectorUtils.java:614`
- OIDC envs are pulled into step envs: `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:2458`
- OIDC envs are merged into ACR step envs early: `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:1080`

Manual credentials are materialized as secrets:
- Manual Azure secret mapping into env vars: `933-ci-commons/src/main/java/io/harness/connector/ConnectorEnvVariablesHelper.java:378`

### Auth Flow Details

```
Auth Method: Manual Credentials (Secret Key)
├── Connector Fields Used: clientId, tenantId, auth.type=Secret, secretRef
├── Mapped to Plugin Env/Args: CLIENT_ID, TENANT_ID, CLIENT_SECRET
└── Code Location: env var names `PluginSettingUtils.java:705`; secret values `ConnectorEnvVariablesHelper.java:384`
```

```
Auth Method: Manual Credentials (Certificate)
├── Connector Fields Used: clientId, tenantId, auth.type=Certificate, certificateRef
├── Mapped to Plugin Env/Args: CLIENT_ID, TENANT_ID, CLIENT_CERTIFICATE
└── Code Location: env var names `PluginSettingUtils.java:705`; secret values `ConnectorEnvVariablesHelper.java:410`
```

```
Auth Method: OIDC / Workload Identity
├── Connector Fields Used: tenantId, clientId, audience
├── Mapped to Plugin Env/Args: AZURE_CLIENT_ID, AZURE_TENANT_ID, AZURE_AUDIENCE, PLUGIN_OIDC_TOKEN_ID
└── Code Location: env map creation `AzureOidcAuthenticator.java:69`; injected `PluginSettingUtils.java:2458`
```

```
Auth Method: Inherit From Delegate (Managed Identity)
├── Connector Fields Used: auth.type=SystemAssignedManagedIdentity or UserAssignedManagedIdentity; user-assigned requires clientId
├── Mapped to Plugin Env/Args: none directly (delegate identity is used)
└── Code Location: spec `AzureInheritFromDelegateDetailsDTO.java:36`; user-assigned clientId `AzureUserAssignedMSIAuthDTO.java:31`
```

## Phase 4: Plugin Inputs Summary (All Inputs)

Primary ACR env assembly:
- `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:1077`

Plugin images configured for ACR:
- Kaniko ACR: `332-ci-manager/config/ci-manager-config.yml:396`
- Buildx ACR: `332-ci-manager/config/ci-manager-config.yml:428`

### Step Inputs → Plugin Env Vars

| Input Name | Type | Required | Description | Maps To (Plugin) |
|---|---|---:|---|---|
| connectorRef | string | Yes | Azure connector reference | Connector-derived envs (see auth flow matrix); mapping defined at `PluginSettingUtils.java:705` |
| repository | string | Yes | Full ACR repository, including registry | `PLUGIN_REPO` and `PLUGIN_REGISTRY` (`PluginSettingUtils.java:1085`) |
| tags | list<string> | Yes | Image tags | `PLUGIN_TAGS` (`PluginSettingUtils.java:1095`) |
| subscriptionId | string | No | Azure subscription ID (artifact URL helper) | `SUBSCRIPTION_ID` (`PluginSettingUtils.java:1086`); used for artifact URL (`ACRStep.java:119`) |
| dockerfile | string | No | Dockerfile path | `PLUGIN_DOCKERFILE` (`PluginSettingUtils.java:1098`) |
| context | string | No | Build context | `PLUGIN_CONTEXT` (`PluginSettingUtils.java:1104`) |
| target | string | No | Docker target stage | `PLUGIN_TARGET` (`PluginSettingUtils.java:1109`) |
| buildArgs | map<string,string> | No | Docker build args | `PLUGIN_BUILD_ARGS`, `PLUGIN_BUILD_ARGS_NEW` (`PluginSettingUtils.java:1114`) |
| labels | map<string,string> | No | Image labels | `PLUGIN_CUSTOM_LABELS` (`PluginSettingUtils.java:1121`) |
| optimize | boolean | No | K8 snapshot optimization (defaults true) | `PLUGIN_SNAPSHOT_MODE=redo` on K8 (`PluginSettingUtils.java:1173`) |
| remoteCacheImage | string | No | Remote cache image | K8: `PLUGIN_ENABLE_CACHE` + `PLUGIN_CACHE_REPO` or appended to `PLUGIN_CACHE_FROM` (`PluginSettingUtils.java:1178`) |
| baseImageConnectorRefs | list<string> | No | Base image connector(s) | `PLUGIN_BASE_IMAGE_REGISTRY` (`PluginSettingUtils.java:770`) and base image secrets `PLUGIN_BASE_IMAGE_USERNAME/PASSWORD` (`PluginSettingUtils.java:801`) |
| cacheFrom | list<string> | No | Cache sources | `PLUGIN_CACHE_FROM` (`PluginSettingUtils.java:1161` and `PluginSettingUtils.java:1192`) |
| cacheTo | string | No | Cache destination | `PLUGIN_CACHE_TO` (`PluginSettingUtils.java:1135`) |
| caching | boolean | No | Enables buildx/DLC behavior | `PLUGIN_BUILDER_DRIVER_OPTS(_NEW)` and cache metrics (`PluginSettingUtils.java:1139`) |
| envDockerSecrets | map<string,string> | No | Docker secrets via env (buildx only) | `PLUGIN_SECRETS_FROM_ENV` (`PluginSettingUtils.java:1463`) |
| fileDockerSecrets | map<string,string> | No | Docker secrets via file (buildx only) | `PLUGIN_SECRETS_FROM_FILE` (`PluginSettingUtils.java:1463`) |
| envVariables | map<string,string> | No | Extra env vars | Merged directly (`PluginSettingUtils.java:1166`) |
| runAsUser | integer | No | Container user override | Runtime container setting (`VmPluginCompatibleStepSerializer.java:237`) |
| resources | object | No | Container resources | Runtime container setting (`ACRStepInfo.java:84`) |
| retry | integer | No | Step retry count | Defaults to 1 (`ACRStepInfo.java:59`) |

### Auth Flow Matrix

| Auth Method | Connector Fields | Plugin Env Vars | Notes |
|---|---|---|---|
| Manual Credentials (Secret Key) | `clientId`, `tenantId`, `auth.type=Secret`, `secretRef` | `CLIENT_ID`, `TENANT_ID`, `CLIENT_SECRET` | Env names from `PluginSettingUtils.java:705`; values from `ConnectorEnvVariablesHelper.java:384` |
| Manual Credentials (Certificate) | `clientId`, `tenantId`, `auth.type=Certificate`, `certificateRef` | `CLIENT_ID`, `TENANT_ID`, `CLIENT_CERTIFICATE` | Env names from `PluginSettingUtils.java:705`; values from `ConnectorEnvVariablesHelper.java:410` |
| OIDC | `tenantId`, `clientId`, `audience` | `AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, `AZURE_AUDIENCE`, `PLUGIN_OIDC_TOKEN_ID` | Created in `AzureOidcAuthenticator.java:69`; injected via `PluginSettingUtils.java:2458` |
| Inherit From Delegate (MSI) | `auth.type=SystemAssignedManagedIdentity` or `UserAssignedManagedIdentity`; user-assigned requires `clientId` | None directly | Delegate identity is used; no env map is injected (`BaseConnectorUtils.java:611`) |

## Infra Applicability (K8 vs VM vs Hosted/DLITE)

Infra types for CI steps are:
- `K8`, `VM`, and `DLITE_VM`: `879-pipeline-ci-commons/src/main/java/io/harness/beans/sweepingoutputs/StageInfraDetails.java:33`

K8 flows call the plugin env builder with `Type.K8`:
- `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/serializer/PluginCompatibleStepSerializer.java:76`

VM and hosted/DLITE flows use the VM serializer path:
- `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/serializer/vm/VmPluginCompatibleStepSerializer.java:212`

The ACR step’s env assembly explicitly branches on infra type:
- `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:1153`

### Applicability Matrix

| Concern | K8 | VM | Hosted/DLITE_VM | Notes |
|---|---:|---:|---:|---|
| Connector auth env name mapping (Azure manual) | Yes | Yes | Yes (via VM path) | Central mapping: `PluginSettingUtils.java:705`; K8 uses it: `K8InitializeStepUtils.java:2220`; VM merges it: `VmPluginCompatibleStepSerializer.java:267` |
| OIDC env injection (`AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, `AZURE_AUDIENCE`, `PLUGIN_OIDC_TOKEN_ID`) | Yes | Yes | Yes (via VM path) | OIDC injected before step envs: `PluginSettingUtils.java:1080`, `PluginSettingUtils.java:2458` |
| Core ACR plugin envs (`PLUGIN_REGISTRY`, `PLUGIN_REPO`, `PLUGIN_TAGS`, etc.) | Yes | Yes | Yes (via VM path) | Common assembly in `getACRStepInfoEnvVariables`: `PluginSettingUtils.java:1077` |
| K8-only envs/behaviors | Yes | No | No | K8 branch: `PluginSettingUtils.java:1153`; snapshot mode: `PluginSettingUtils.java:1173`; artifact file: `PluginSettingUtils.java:1195` |
| VM-only envs/behaviors | No | Yes | Yes (via VM path) | VM branch: `PluginSettingUtils.java:1158`; daemon off: `PluginSettingUtils.java:1159` |

## UI Field / Input

| Field / input | KubernetesDirect | KubernetesHosted | Cloud | VM | Docker | Optional? | Feature-flag impact | License impact | Tooltip ID | Tooltip content |
|---|---:|---:|---:|---:|---:|---|---|---|---|---|
| `identifier` | Y | Y | Y | Y | Y | No (Required) | None | None | None | None |
| `name` | Y | Y | Y | Y | Y | No (Required) | None | None | None | None |
| `spec.connectorRef` | Y | Y | Y | Y | Y | No (Required) | None | None | None *(Edit)* / `acrConnector` *(Input Set)* | `acrConnector` content not found in tooltip dictionary bundle |
| `spec.repository` | Y | Y | Y | Y | Y | No (Required) | None | None | `repository` | Not found in tooltip dictionary bundle |
| `spec.subscriptionId` | Y | Y | Y | Y | Y | Yes | None | None | `subscriptionId` | Not found in tooltip dictionary bundle |
| `spec.tags` | Y | Y | Y | Y | Y | No (Required) | None | None | None *(Edit)* | None |
| `spec.caching` | Y | Y | Y | Y | Y | Yes | `CI_ENABLE_INTELLIGENT_DEFAULTS` may default new step to `true` when unset | None | None | None |
| `spec.baseImageConnectorRefs` | Y | Y | Y | Y | Y | Yes | None | None | `baseImageConnectorRefs` | Select authenticated Docker connector for base image pulls; recommended to avoid unauthenticated rate limits |
| `spec.optimize` | Y | Y | N | N | N | Yes | None | None | `optimize` | Enables `--snapshotMode=redo` to reduce snapshot time |
| `spec.dockerfile` | Y | Y | Y | Y | Y | Yes | None | None | `dockerfile` | Dockerfile name; defaults to root if not provided |
| `spec.context` | Y | Y | Y | Y | Y | Yes | None | None | `context` | Path to Docker build context used by build (e.g., `COPY`) |
| `spec.labels` | Y | Y | Y | Y | Y | Yes | None | None | None | None |
| `spec.buildArgs` | Y | Y | Y | Y | Y | Yes | None | None | None | None |
| `spec.target` | Y | Y | Y | Y | Y | Yes | None | None | `target` | Docker target build stage from Dockerfile |
| `spec.remoteCacheImage` | Y | Y | N | N | N | Yes | None | None | `gcrRemoteCache` | Remote Docker layer cache image; repo should be in same host/project as build image |
| `spec.envVariables` | Y | Y | Y | Y | Y | Yes | None | None | None | None |
| `spec.runAsUser` | Y | N | Y | N | N | Yes | None | None | `runAsUser` | User ID for running processes in pod/container security context |
| `spec.limitMemory` | Y | Y | N | N | N | Yes | None | None | `limitMemory` | Max memory; examples include `Mi/Gi`, default `500Mi` |
| `spec.limitCPU` | Y | Y | N | N | N | Yes | None | None | `limitCPULabel` | Max CPU cores/millicpu, default `400m` |
| `timeout` | Y | Y | Y | Y | Y | Yes | None | None | None *(Edit)* | None |

## Key Takeaways

- ACR connector auth mappings are applied on both K8 and VM/hosted flows, but OIDC and manual flows use different env var names.
- The step’s plugin env inputs are mostly shared, with infra-specific branches for caching and artifact metadata.
- `repository` is the single source of truth for registry and repo; `subscriptionId` only affects artifact URLs.
