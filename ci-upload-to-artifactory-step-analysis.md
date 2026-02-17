# Upload Artifacts to JFrog Artifactory: Inputs & Connector Mapping

This document traces the CI Manager implementation of the **Upload Artifacts to JFrog Artifactory** step, its user-facing inputs, Artifactory connector auth flows, connector → plugin env mapping, and infra-type applicability.

All code locations below are workspace-relative `path:line` references.

type: ArtifactoryUpload
Plugin used: plugins/artifactory:1.8.3 (K8)

## Known Gaps / Gotchas

- `uploadAsFlat` defaults to `true` when not set.
- `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:2021`
- Connector proxy is only enabled on VM infra.
- `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:2114`
- Anonymous auth does not populate connector env vars in `ConnectorEnvVariablesHelper`.
- `933-ci-commons/src/main/java/io/harness/connector/ConnectorEnvVariablesHelper.java:72`

## Phase 1: Step Definition & Executor

Step registration and display metadata:
- `332-ci-manager/service/src/main/java/io/harness/ci/execution/plan/creator/CIPipelineServiceInfoProvider.java:352`

Step node wiring to the spec:
- `332-ci-manager/service/src/main/java/io/harness/ci/beans/steps/nodes/ArtifactoryUploadNode.java:35`
- `332-ci-manager/service/src/main/java/io/harness/ci/beans/steps/nodes/ArtifactoryUploadNode.java:44`

Plan creator:
- `332-ci-manager/service/src/main/java/io/harness/ci/execution/plancreator/ArtifactoryUploadStepPlanCreator.java:19`

Executor (engine step):
- `332-ci-manager/service/src/main/java/io/harness/ci/execution/registrars/ExecutionRegistrar.java:104`
- `332-ci-manager/service/src/main/java/io/harness/ci/execution/states/UploadToArtifactoryStep.java:16`

User-facing spec model (inputs live here):
- `879-pipeline-ci-commons/src/main/java/io/harness/beans/steps/stepinfo/UploadToArtifactoryStepInfo.java:53`

Default retry value:
- `879-pipeline-ci-commons/src/main/java/io/harness/beans/steps/stepinfo/UploadToArtifactoryStepInfo.java:54`

Step env assembly entry point for Artifactory upload:
- `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:341`
- `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:2013`

## Phase 2: Artifactory Connector Auth Methods & Required Fields

Supported Artifactory auth types:
- `954-connector-beans/src/main/java/io/harness/delegate/beans/connector/artifactoryconnector/ArtifactoryAuthType.java:18`

Required fields by auth method:

Username + Password:
- Required fields: `passwordRef`, and `username` or `usernameRef`.
- `954-connector-beans/src/main/java/io/harness/delegate/beans/connector/artifactoryconnector/ArtifactoryUsernamePasswordAuthDTO.java:36`

Anonymous:
- No credentials required.
- `954-connector-beans/src/main/java/io/harness/delegate/beans/connector/ArtifactoryConnectorDTO.java:86`

Connector auth block:
- `954-connector-beans/src/main/java/io/harness/delegate/beans/connector/artifactoryconnector/ArtifactoryAuthenticationDTO.java:38`

## Phase 3: Connector → Plugin Env Mapping (Per Auth Flow)

### Mapping Chain Overview

Connector env var names are decided by step type:
- Artifactory secret env name mapping: `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:733`

The mapping is applied on both K8 and VM paths:
- K8 conversion info uses the mapping: `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/integrationstage/k8s/K8InitializeStepUtils.java:2220`
- VM serializer merges the mapping into connector details: `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/serializer/vm/VmPluginCompatibleStepSerializer.java:267`

Connector secrets are materialized here:
- `933-ci-commons/src/main/java/io/harness/connector/ConnectorEnvVariablesHelper.java:72`

### Auth Flow Details

```
Auth Method: Username + Password
├── Connector Fields Used: username or usernameRef, passwordRef, artifactoryServerUrl
├── Mapped to Plugin Env/Args: PLUGIN_USERNAME, PLUGIN_PASSWORD, PLUGIN_URL
└── Code Location: env var names `PluginSettingUtils.java:733`; secret values `ConnectorEnvVariablesHelper.java:84`
```

```
Auth Method: Anonymous
├── Connector Fields Used: none
├── Mapped to Plugin Env/Args: none directly
└── Code Location: auth type `ArtifactoryAuthType.java:18`
```

## Phase 4: Plugin Inputs Summary (All Inputs)

Primary Artifactory upload env assembly:
- `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:2013`

Plugin images configured for Artifactory upload:
- `332-ci-manager/config/ci-manager-config.yml:453`

### Step Inputs → Plugin Env Vars

| Input Name | Type | Required | Description | Maps To (Plugin) |
|---|---|---:|---|---|
| connectorRef | string | Yes | Artifactory connector reference | Connector-derived envs (see auth flow matrix); mapping defined at `PluginSettingUtils.java:733` |
| sourcePath | string | Yes | Local path to upload | `PLUGIN_SOURCE` (`PluginSettingUtils.java:2017`) |
| target | string | Yes | Artifactory target path | `PLUGIN_TARGET` (`PluginSettingUtils.java:2019`) |
| uploadAsFlat | boolean | No | Upload as flat structure | `PLUGIN_FLAT` (`PluginSettingUtils.java:2021`) |
| runAsUser | integer | No | Container user override | Runtime container setting (`VmPluginCompatibleStepSerializer.java:237`) |
| resources | object | No | Container resources | Runtime container setting (`UploadToArtifactoryStepInfo.java:72`) |
| retry | integer | No | Step retry count | Defaults to 1 (`UploadToArtifactoryStepInfo.java:54`) |

### Auth Flow Matrix

| Auth Method | Connector Fields | Plugin Env Vars | Notes |
|---|---|---|---|
| Username + Password | `username` or `usernameRef`, `passwordRef`, `artifactoryServerUrl` | `PLUGIN_USERNAME`, `PLUGIN_PASSWORD`, `PLUGIN_URL` | Env names from `PluginSettingUtils.java:733`; values from `ConnectorEnvVariablesHelper.java:84` |
| Anonymous | none | none directly | No envs populated by `ConnectorEnvVariablesHelper.java:72` |

## Infra Applicability (K8 vs VM vs Hosted/DLITE)

Infra types for CI steps are:
- `K8`, `VM`, and `DLITE_VM`: `879-pipeline-ci-commons/src/main/java/io/harness/beans/sweepingoutputs/StageInfraDetails.java:33`

K8 flows call the plugin env builder with `Type.K8`:
- `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/serializer/PluginCompatibleStepSerializer.java:76`

VM and hosted/DLITE flows use the VM serializer path:
- `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/serializer/vm/VmPluginCompatibleStepSerializer.java:90`

Proxy env is only set on VM:
- `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:2114`

### Applicability Matrix

| Concern | K8 | VM | Hosted/DLITE_VM | Notes |
|---|---:|---:|---:|---|
| Connector auth env name mapping | Yes | Yes | Yes (via VM path) | Central mapping: `PluginSettingUtils.java:733`; K8 uses it: `K8InitializeStepUtils.java:2220`; VM merges it: `VmPluginCompatibleStepSerializer.java:267` |
| Core upload plugin envs (`PLUGIN_SOURCE`, `PLUGIN_TARGET`, `PLUGIN_FLAT`) | Yes | Yes | Yes (via VM path) | Assembly in `getUploadToArtifactoryStepInfoEnvVariables`: `PluginSettingUtils.java:2013` |
| Proxy env (`PLUGIN_ENABLE_PROXY`) | No | Yes | Yes (via VM path) | Proxy only for VM: `PluginSettingUtils.java:2114` |

## UI Field / Input

Build flows considered: `Cloud`, `KubernetesDirect`, `KubernetesHosted`, `VM`, `Docker`.

| Field / Input | Visible In Build Flow(s) | Optional? | Tooltip ID | Tooltip Text / Content | Placeholder | Feature-Flag Influence | License Influence |
|---|---|---|---|---|---|---|---|
| `identifier` | All | No | — | — | — | None | None |
| `name` | All | No | `jfrogArt_name` | Enter a name for this step. (Includes step help text/link for Upload Artifacts to JFrog Artifactory.) | — | None | None |
| `description` | All | Yes | `jfrogArt_description` | Not found in shared tooltip dictionary (ID is rendered in UI). | — | None | None |
| `spec.connectorRef` | All | No | `jfrogArt_spec.connectorRef` | Select the Harness Artifactory Connector to use for upload. | `select` | None | None |
| `spec.target` | All | No | `jFrogArtifactoryTarget` | Repository name relative to connector server URL; if no `pom.xml`, target should be full artifacts folder path. | — | None | None |
| `spec.sourcePath` | All | No | `sourcePath` | Path to artifact file/folder to upload. | — | None | None |
| `spec.uploadAsFlat` (optional config) | All (when FF enabled) | Yes | `uploadAsFlat` | If false, upload preserves filesystem hierarchy; default behavior is flat upload. | `select` | Controlled by `CI_ENABLE_UPLOAD_AS_FLAT` | None |
| `spec.runAsUser` (optional config) | `Cloud`, `KubernetesDirect` | Yes | `runAsUser` | UID used to run processes in pod/containers. | `1000` | None | None |
| `spec.limitMemory` (optional config) | `KubernetesDirect`, `KubernetesHosted` | Yes | `limitMemory` | Max container memory (default `500Mi`). | — | None | None |
| `spec.limitCPU` (optional config) | `KubernetesDirect`, `KubernetesHosted` | Yes | `limitCPULabel` | Max container CPU (default `400m`). | — | None | None |
| `timeout` (optional config) | All | Yes | — (edit view) | — | `common.durationPlaceholder` | None | None |

If you want, I can do the same full table next for **Build and Push to ECR** or any other CI step.

## Key Takeaways

- Artifactory Upload uses the Artifactory connector with either Username/Password or Anonymous auth.
- The step maps `sourcePath`, `target`, and `uploadAsFlat` into the plugin envs, and injects Artifactory connector env vars when credentials are present.
- Proxy support exists only for VM/hosted execution, not K8.
