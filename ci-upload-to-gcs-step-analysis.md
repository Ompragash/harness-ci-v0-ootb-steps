# Upload to GCS: Inputs & GCP Connector Mapping

This document traces the CI Manager implementation of the **Upload Artifacts to GCS** step, its user-facing inputs, GCP connector auth flows, connector → plugin env mapping, and infra-type applicability.

All code locations below are workspace-relative `path:line` references.

type: GCSUpload
Plugin used: plugins/gcs:1.6.6 (K8), github.com/drone-plugins/drone-gcs@refs/tags/v1.6.6 (VM containerless)

## Known Gaps / Gotchas

- The step input `sourcePath` is resolved using the parameter name `sourcePaths` (plural) in the env builder.
- `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:2134`
- The `bucket` input is not passed as a standalone env var. `bucket` and optional `target` are combined into `PLUGIN_TARGET`.
- `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:2137`
- Proxy support is only enabled on VM infra when the connector uses a proxy.
- `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:2114`
- OIDC env vars use `PLUGIN_*` fields (same as GAR), not standard Google env names.
- `879-pipeline-ci-commons/src/main/java/io/harness/utils/GcpOidcAuthenticator.java:99`
- There is no separate “Cloud” infra type; hosted/DLITE flows go through the VM serializer path.
- Infra types: `879-pipeline-ci-commons/src/main/java/io/harness/beans/sweepingoutputs/StageInfraDetails.java:33`
- VM serializer path: `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/serializer/vm/VmPluginCompatibleStepSerializer.java:212`

## Phase 1: Step Definition & Executor

Step registration and display metadata:
- `332-ci-manager/service/src/main/java/io/harness/ci/execution/plan/creator/CIPipelineServiceInfoProvider.java:381`

Step node wiring to the spec:
- `332-ci-manager/service/src/main/java/io/harness/ci/beans/steps/nodes/GCSUploadNode.java:35`
- `332-ci-manager/service/src/main/java/io/harness/ci/beans/steps/nodes/GCSUploadNode.java:45`

Plan creator:
- `332-ci-manager/service/src/main/java/io/harness/ci/execution/plancreator/GCSUploadStepPlanCreator.java:19`

Executor (engine step):
- `332-ci-manager/service/src/main/java/io/harness/ci/execution/registrars/ExecutionRegistrar.java:99`
- `332-ci-manager/service/src/main/java/io/harness/ci/execution/states/UploadToGCSStep.java:25`

User-facing spec model (inputs live here):
- `879-pipeline-ci-commons/src/main/java/io/harness/beans/steps/stepinfo/UploadToGCSStepInfo.java:53`

Default retry and timeout values:
- `879-pipeline-ci-commons/src/main/java/io/harness/beans/steps/stepinfo/UploadToGCSStepInfo.java:54`
- `879-pipeline-ci-commons/src/main/java/io/harness/beans/steps/stepinfo/UploadToGCSStepInfo.java:55`

Step env assembly entry point for GCS upload:
- `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:346`
- `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:2126`

## Phase 2: GCP Connector Auth Methods & Required Fields

Supported GCP credential types:
- `954-connector-beans/src/main/java/io/harness/delegate/beans/connector/gcpconnector/GcpCredentialType.java:13`

Required fields by auth method:

Manual credentials:
- Required field: `secretKeyRef`.
- `954-connector-beans/src/main/java/io/harness/delegate/beans/connector/gcpconnector/GcpManualDetailsDTO.java:31`

OIDC:
- Required fields: `workloadPoolId`, `providerId`, `gcpProjectId`, `serviceAccountEmail`.
- `954-connector-beans/src/main/java/io/harness/delegate/beans/connector/gcpconnector/GcpOidcDetailsDTO.java:29`

Inherit from delegate:
- Required field: `delegateSelectors`.
- `954-connector-beans/src/main/java/io/harness/delegate/beans/connector/gcpconnector/GcpDelegateDetailsDTO.java:27`
- Validation enforces selectors for inherit-from-delegate.
- `954-connector-beans/src/main/java/io/harness/delegate/beans/connector/GcpConnectorDTO.java:81`

Connector detail assembly by auth type:
- `879-pipeline-ci-commons/src/main/java/io/harness/ci/utils/BaseConnectorUtils.java:645`

## Phase 3: Connector → Plugin Env Mapping (Per Auth Flow)

### Mapping Chain Overview

Connector env var names are decided by step type:
- GCS upload secret env name mapping: `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:711`

The mapping is applied on both K8 and VM paths:
- K8 conversion info uses the mapping: `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/integrationstage/k8s/K8InitializeStepUtils.java:2220`
- VM serializer merges the mapping into connector details: `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/serializer/vm/VmPluginCompatibleStepSerializer.java:267`

OIDC values are generated and injected through connector details:
- OIDC env map creation: `879-pipeline-ci-commons/src/main/java/io/harness/utils/GcpOidcAuthenticator.java:99`
- OIDC map attached to connector details: `879-pipeline-ci-commons/src/main/java/io/harness/ci/utils/BaseConnectorUtils.java:659`
- OIDC envs are pulled into step envs: `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:2458`
- OIDC envs are merged into GCS upload step envs early: `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:2129`

Manual credentials are materialized as secrets:
- Manual GCP secret mapping into env vars: `933-ci-commons/src/main/java/io/harness/connector/ConnectorEnvVariablesHelper.java:115`

### Auth Flow Details

```
Auth Method: Manual Credentials (Service Account Key)
├── Connector Fields Used: secretKeyRef
├── Mapped to Plugin Env/Args: PLUGIN_JSON_KEY
└── Code Location: env var names `PluginSettingUtils.java:711`; secret values `ConnectorEnvVariablesHelper.java:115`
```

```
Auth Method: OIDC / Workload Identity
├── Connector Fields Used: workloadPoolId, providerId, gcpProjectId, serviceAccountEmail
├── Mapped to Plugin Env/Args: PLUGIN_PROJECT_NUMBER, PLUGIN_PROVIDER_ID, PLUGIN_POOL_ID, PLUGIN_SERVICE_ACCOUNT_EMAIL, PLUGIN_OIDC_TOKEN_ID
└── Code Location: env map creation `GcpOidcAuthenticator.java:99`; injected `PluginSettingUtils.java:2458`
```

```
Auth Method: Inherit From Delegate
├── Connector Fields Used: delegateSelectors
├── Mapped to Plugin Env/Args: none directly (delegate identity is used)
└── Code Location: spec `GcpDelegateDetailsDTO.java:27`; validation `GcpConnectorDTO.java:81`
```

## Phase 4: Plugin Inputs Summary (All Inputs)

Primary GCS upload env assembly:
- `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:2126`

Plugin images configured for GCS upload:
- `332-ci-manager/config/ci-manager-config.yml:435`

### Step Inputs → Plugin Env Vars

| Input Name | Type | Required | Description | Maps To (Plugin) |
|---|---|---:|---|---|
| connectorRef | string | Yes | GCP connector reference | Connector-derived envs (see auth flow matrix); mapping defined at `PluginSettingUtils.java:711` |
| bucket | string | Yes | GCS bucket name | Combined into `PLUGIN_TARGET` with `target` (`PluginSettingUtils.java:2137`) |
| sourcePath | string | Yes | Local path to upload | `PLUGIN_SOURCE` (`PluginSettingUtils.java:2134`) |
| target | string | No | Object prefix within bucket | Combined into `PLUGIN_TARGET` (`PluginSettingUtils.java:2140`) |
| runAsUser | integer | No | Container user override | Runtime container setting (`VmPluginCompatibleStepSerializer.java:237`) |
| resources | object | No | Container resources | Runtime container setting (`UploadToGCSStepInfo.java:73`) |
| retry | integer | No | Step retry count | Defaults to 1 (`UploadToGCSStepInfo.java:54`) |

### Auth Flow Matrix

| Auth Method | Connector Fields | Plugin Env Vars | Notes |
|---|---|---|---|
| Manual Credentials | `secretKeyRef` | `PLUGIN_JSON_KEY` | Env names from `PluginSettingUtils.java:711`; values from `ConnectorEnvVariablesHelper.java:115` |
| OIDC / Workload Identity | `workloadPoolId`, `providerId`, `gcpProjectId`, `serviceAccountEmail` | `PLUGIN_PROJECT_NUMBER`, `PLUGIN_PROVIDER_ID`, `PLUGIN_POOL_ID`, `PLUGIN_SERVICE_ACCOUNT_EMAIL`, `PLUGIN_OIDC_TOKEN_ID` | Created in `GcpOidcAuthenticator.java:99`; injected via `PluginSettingUtils.java:2458` |
| Inherit From Delegate | `delegateSelectors` | None directly | Selectors required: `GcpDelegateDetailsDTO.java:27`; enforced: `GcpConnectorDTO.java:81` |

## Infra Applicability (K8 vs VM vs Hosted/DLITE)

Infra types for CI steps are:
- `K8`, `VM`, and `DLITE_VM`: `879-pipeline-ci-commons/src/main/java/io/harness/beans/sweepingoutputs/StageInfraDetails.java:33`

K8 flows call the plugin env builder with `Type.K8`:
- `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/serializer/PluginCompatibleStepSerializer.java:76`

VM and hosted/DLITE flows use the VM serializer path:
- `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/serializer/vm/VmPluginCompatibleStepSerializer.java:212`

The GCS upload step’s env assembly is shared across infra types but enables proxy only on VM:
- `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:2114`

### Applicability Matrix

| Concern | K8 | VM | Hosted/DLITE_VM | Notes |
|---|---:|---:|---:|---|
| Connector auth env name mapping (`GCP_KEY` → `PLUGIN_JSON_KEY`) | Yes | Yes | Yes (via VM path) | Central mapping: `PluginSettingUtils.java:711`; K8 uses it: `K8InitializeStepUtils.java:2220`; VM merges it: `VmPluginCompatibleStepSerializer.java:267` |
| OIDC env injection (`PLUGIN_*` OIDC fields) | Yes | Yes | Yes (via VM path) | OIDC injected before step envs: `PluginSettingUtils.java:2129`, `PluginSettingUtils.java:2458` |
| Core upload plugin envs (`PLUGIN_SOURCE`, `PLUGIN_TARGET`) | Yes | Yes | Yes (via VM path) | Assembly in `getUploadToGCSStepInfoEnvVariables`: `PluginSettingUtils.java:2126` |
| Proxy env (`PLUGIN_ENABLE_PROXY`) | No | Yes | Yes (via VM path) | Proxy only for VM: `PluginSettingUtils.java:2114` |

## UI Field / Input

| Field / Input | Visible In Build Flow(s) | Optional? | Tooltip ID | Tooltip Text / Content | Placeholder | Feature-Flag Influence | License Influence |
|---|---|---|---|---|---|---|---|
| `identifier` | All | No | — | — | — | None | None |
| `name` | All | No | `ciGcsStep_name` | Enter a name for the Step. | — | None | None |
| `spec.connectorRef` | All | No | `ciGcsStep_spec.connectorRef` | Harness GCP Connector for the GCP account where artifact is uploaded. | `select` | None | None |
| `spec.bucket` | All | No | `gcsBucket` | Enter the source GCS bucket name. | — | None | None |
| `spec.sourcePath` | All | No | `sourcePath` | Path to the artifact file/folder to upload. | — | None | None |
| `spec.target` (optional config) | All | Yes | `gcsS3Target` | Path relative to bucket where artifact is stored. | `pipelineSteps.artifactsTargetPlaceholder` | None | None |
| `spec.runAsUser` | `Cloud`, `KubernetesDirect` | Yes | `runAsUser` | UID used to run processes in pod/containers. | `1000` | None | None |
| `spec.limitMemory` | `KubernetesDirect`, `KubernetesHosted` | Yes | `limitMemory` | Max container memory (default `500Mi`). | — | None | None |
| `spec.limitCPU` | `KubernetesDirect`, `KubernetesHosted` | Yes | `limitCPULabel` | Max container CPU (default `400m`). | — | None | None |
| `timeout` | All | Yes | — (edit view) | — | `common.durationPlaceholder` | None | None |

Notes:
- `spec.target` is in the step’s **Optional Config** section.
- Runtime/input-set paths can show `<+input>` placeholder for runtime-enabled fields.

## Key Takeaways

- Upload to GCS uses the GCP connector with the same manual/OIDC/delegate auth flows as GAR.
- The `bucket` and `target` inputs are merged into a single `PLUGIN_TARGET` value; the bucket is not passed separately.
- OIDC env variables are injected for GCP when the connector uses OIDC and are available across K8 and VM/hosted paths.
