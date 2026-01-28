# Save Cache to GCS: Inputs & GCP Connector Mapping

This document traces the CI Manager implementation of the **Save Cache to GCS** step, its user-facing inputs, GCP connector auth flows, connector → plugin env mapping, and infra-type applicability.

All code locations below are workspace-relative `path:line` references.

type: SaveCacheGCS
Plugin used: plugins/cache:1.9.17 (K8), github.com/drone-plugins/drone-meltwater-cache@refs/tags/v1.9.17 (VM containerless)

## Known Gaps / Gotchas

- `key` and `sourcePaths` are `@NotNull` in the spec, but become optional when the step identifier is the implicit Harness cache step (`save-cache-harness`).
  - Spec fields: `879-pipeline-ci-commons/src/main/java/io/harness/beans/steps/stepinfo/SaveCacheGCSStepInfo.java:81`
  - Optional handling for `save-cache-harness`: `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:1840`
  - Implicit step ID constant: `879-pipeline-ci-commons/src/main/java/io/harness/ci/commonconstants/CIExecutionConstants.java:38`
- The Save Cache GCS env builder does not set any proxy-related plugin envs.
  - Env builder: `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:1832`
- There is no separate “Cloud” infra type; hosted/DLITE flows go through the VM serializer path.
  - Infra types: `879-pipeline-ci-commons/src/main/java/io/harness/beans/sweepingoutputs/StageInfraDetails.java:33`
  - VM serializer path: `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/serializer/vm/VmPluginCompatibleStepSerializer.java:112`

## Phase 1: Step Definition & Executor

Step registration and display metadata:
- `332-ci-manager/service/src/main/java/io/harness/ci/execution/plan/creator/CIPipelineServiceInfoProvider.java:306`

Step node wiring to the spec:
- `877-pipeline-ci-cd-commons/src/java/io/harness/beans/steps/nodes/SaveCacheGCSNode.java:41`
- `877-pipeline-ci-cd-commons/src/java/io/harness/beans/steps/nodes/SaveCacheGCSNode.java:51`

Plan creator:
- `332-ci-manager/service/src/main/java/io/harness/ci/execution/plancreator/SaveCacheGCSStepPlanCreator.java:19`

Executor (engine step):
- `332-ci-manager/service/src/main/java/io/harness/ci/execution/registrars/ExecutionRegistrar.java:100`
- `332-ci-manager/service/src/main/java/io/harness/ci/execution/states/SaveCacheGCSStep.java:16`

User-facing spec model (inputs live here):
- `879-pipeline-ci-commons/src/main/java/io/harness/beans/steps/stepinfo/SaveCacheGCSStepInfo.java:52`

Default retry value:
- `879-pipeline-ci-commons/src/main/java/io/harness/beans/steps/stepinfo/SaveCacheGCSStepInfo.java:59`

Step env assembly entry point for Save Cache to GCS:
- `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:351`
- `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:1832`

Plugin images configured for Save Cache to GCS:
- K8 plugin image: `332-ci-manager/config/ci-manager-config.yml:465`
- VM containerless plugin: `332-ci-manager/config/ci-manager-config.yml:676`

## Phase 2: GCP Connector Auth Methods & Required Fields

Supported GCP credential types:
- `954-connector-beans/src/main/java/io/harness/delegate/beans/connector/gcp/GcpCredentialType.java:20`

Required fields by auth method:

Manual credentials:
- Required field: `serviceAccountKeyRef`.
- `954-connector-beans/src/main/java/io/harness/delegate/beans/connector/gcp/GcpManualDetailsDTO.java:33`

OIDC:
- Required field: `serviceAccountEmail`.
- `954-connector-beans/src/main/java/io/harness/delegate/beans/connector/gcp/GcpOidcSpecDTO.java:27`

Connector detail assembly by auth type:
- `879-pipeline-ci-commons/src/main/java/io/harness/ci/utils/BaseConnectorUtils.java:638`

## Phase 3: Connector → Plugin Env Mapping (Per Auth Flow)

### Mapping Chain Overview

Connector env var names are decided by step type:
- Save Cache GCS secret env name mapping: `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:712`

The mapping is applied on both K8 and VM paths:
- K8 conversion info uses the mapping: `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/integrationstage/k8s/K8InitializeStepUtils.java:2220`
- VM serializer merges the mapping into connector details: `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/serializer/vm/VmPluginCompatibleStepSerializer.java:267`

OIDC values are generated and injected through connector details:
- OIDC env map creation: `879-pipeline-ci-commons/src/main/java/io/harness/utils/GcpOidcAuthenticator.java:58`
- OIDC map attached to connector details: `879-pipeline-ci-commons/src/main/java/io/harness/ci/utils/BaseConnectorUtils.java:658`
- OIDC envs are pulled into step envs: `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:2458`
- OIDC envs are merged into Save Cache GCS step envs early: `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:1835`

Manual credentials are materialized as secrets:
- Manual GCP secret mapping into env vars: `933-ci-commons/src/main/java/io/harness/connector/ConnectorEnvVariablesHelper.java:395`

### Auth Flow Details

```
Auth Method: Manual Credentials (Service Account Key)
├── Connector Fields Used: serviceAccountKeyRef
├── Mapped to Plugin Env/Args: PLUGIN_JSON_KEY
└── Code Location: env var names `PluginSettingUtils.java:712`; secret values `ConnectorEnvVariablesHelper.java:395`
```

```
Auth Method: OIDC
├── Connector Fields Used: serviceAccountEmail
├── Mapped to Plugin Env/Args: PLUGIN_OIDC_TOKEN_ID, PLUGIN_GOOGLE_ACCOUNT_EMAIL
└── Code Location: env map creation `GcpOidcAuthenticator.java:58`; injected `PluginSettingUtils.java:2458`
```

## Phase 4: Plugin Inputs Summary (All Inputs)

Primary Save Cache GCS env assembly:
- `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:1832`

### Step Inputs → Plugin Env Vars

| Input Name | Type | Required | Description | Maps To (Plugin) |
|---|---|---:|---|---|
| connectorRef | string | Yes | GCP connector reference | Connector-derived envs (see auth flow matrix) |
| key | string | Yes | Cache key | `PLUGIN_CACHE_KEY` (`PluginSettingUtils.java:1852`) |
| bucket | string | Yes | GCS bucket | `PLUGIN_BUCKET` (`PluginSettingUtils.java:1859`) |
| sourcePaths | list<string> | Yes | Paths to archive | `PLUGIN_MOUNT` (`PluginSettingUtils.java:1854`) |
| override | boolean | No | Overwrite existing cache | `PLUGIN_OVERRIDE` (`PluginSettingUtils.java:1866`) |
| archiveFormat | enum | No | Cache archive format | `PLUGIN_ARCHIVE_FORMAT` (`PluginSettingUtils.java:1862`) |
| runAsUser | integer | No | Container user override | VM runtime setting, not plugin env (`VmPluginCompatibleStepSerializer.java:237`) |
| resources | object | No | Container resources | Container runtime setting (`SaveCacheGCSStepInfo.java:78`) |
| retry | integer | No | Step retry count | Defaults to 1 (`SaveCacheGCSStepInfo.java:59`) |

Additional plugin envs set by the builder:
- `PLUGIN_REBUILD=true`: `PluginSettingUtils.java:1868`
- `PLUGIN_EXIT_CODE=true`: `PluginSettingUtils.java:1869`
- `PLUGIN_BACKEND=gcs`: `PluginSettingUtils.java:1870`
- `PLUGIN_BACKEND_OPERATION_TIMEOUT`: `PluginSettingUtils.java:1871`

### Auth Flow Matrix

| Auth Method | Connector Fields | Plugin Env Vars | Notes |
|---|---|---|---|
| Manual Credentials | `serviceAccountKeyRef` | `PLUGIN_JSON_KEY` | Env names: `PluginSettingUtils.java:712`; values: `ConnectorEnvVariablesHelper.java:395` |
| OIDC | `serviceAccountEmail` | `PLUGIN_OIDC_TOKEN_ID`, `PLUGIN_GOOGLE_ACCOUNT_EMAIL` | Created in `GcpOidcAuthenticator.java:58`; injected via `PluginSettingUtils.java:2458` |

## Infra Applicability (K8 vs VM vs Hosted/DLITE)

Infra types for CI steps are:
- `K8`, `VM`, and `DLITE_VM`: `879-pipeline-ci-commons/src/main/java/io/harness/beans/sweepingoutputs/StageInfraDetails.java:33`

K8 flows call the plugin env builder with `Type.K8`:
- `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/serializer/PluginCompatibleStepSerializer.java:76`

VM and hosted/DLITE flows call it with `Type.VM`:
- `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/serializer/vm/VmPluginCompatibleStepSerializer.java:112`

The Save Cache GCS env builder does not branch on infra type:
- `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:1832`

### Applicability Matrix

| Concern | K8 | VM | Hosted/DLITE_VM | Notes |
|---|---:|---:|---:|---|
| Connector auth env name mapping (GCP key → plugin envs) | Yes | Yes | Yes (via VM path) | Central mapping: `PluginSettingUtils.java:712`; K8 uses it: `K8InitializeStepUtils.java:2220`; VM merges it: `VmPluginCompatibleStepSerializer.java:267` |
| OIDC env injection (`PLUGIN_OIDC_TOKEN_ID`, `PLUGIN_GOOGLE_ACCOUNT_EMAIL`) | Yes | Yes | Yes (via VM path) | OIDC injected before step envs: `PluginSettingUtils.java:1835`, `PluginSettingUtils.java:2458` |
| Core cache plugin envs (`PLUGIN_BUCKET`, `PLUGIN_CACHE_KEY`, `PLUGIN_MOUNT`, etc.) | Yes | Yes | Yes (via VM path) | Assembly in `getSaveCacheGCSStepInfoEnvVariables`: `PluginSettingUtils.java:1832` |

## Key Takeaways

- Save Cache to GCS uses the GCP connector and supports manual and OIDC auth flows.
- Connector auth values are mapped to plugin envs (`PLUGIN_JSON_KEY`, `PLUGIN_OIDC_TOKEN_ID`, `PLUGIN_GOOGLE_ACCOUNT_EMAIL`).
- Cache behavior is configured via `PLUGIN_CACHE_KEY`, `PLUGIN_MOUNT`, `PLUGIN_BUCKET`, `PLUGIN_ARCHIVE_FORMAT`, and related plugin flags in the env builder.
