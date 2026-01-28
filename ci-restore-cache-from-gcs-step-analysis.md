# Restore Cache From GCS: Inputs & GCP Connector Mapping

This document traces the CI Manager implementation of the **Restore Cache From GCS** step, its user-facing inputs, GCP connector auth flows, connector → plugin env mapping, and infra-type applicability.

All code locations below are workspace-relative `path:line` references.

type: RestoreCacheGCS
Plugin used: plugins/cache:1.9.17 (K8), github.com/drone-plugins/drone-meltwater-cache@refs/tags/v1.9.17 (VM containerless)

## Known Gaps / Gotchas

- `key` is `@NotNull` in the spec, but becomes optional when the step identifier is the implicit Harness cache step (`restore-cache-harness`).
  - Spec field: `879-pipeline-ci-commons/src/main/java/io/harness/beans/steps/stepinfo/RestoreCacheGCSStepInfo.java:81`
  - Optional handling for `restore-cache-harness`: `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:1627`
  - Implicit step ID constant: `879-pipeline-ci-commons/src/main/java/io/harness/ci/commonconstants/CIExecutionConstants.java:37`
- `sourcePaths` is optional in the spec and only set into envs for the implicit Harness cache step.
  - Spec field: `879-pipeline-ci-commons/src/main/java/io/harness/beans/steps/stepinfo/RestoreCacheGCSStepInfo.java:83`
  - Env set only when `restore-cache-harness`: `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:1631`
- The Restore Cache GCS env builder does not set any proxy-related plugin envs.
  - Env builder: `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:1619`
- There is no separate “Cloud” infra type; hosted/DLITE flows go through the VM serializer path.
  - Infra types: `879-pipeline-ci-commons/src/main/java/io/harness/beans/sweepingoutputs/StageInfraDetails.java:33`
  - VM serializer path: `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/serializer/vm/VmPluginCompatibleStepSerializer.java:112`

## Phase 1: Step Definition & Executor

Step registration and display metadata:
- `332-ci-manager/service/src/main/java/io/harness/ci/execution/plan/creator/CIPipelineServiceInfoProvider.java:288`

Step node wiring to the spec:
- `877-pipeline-ci-cd-commons/src/java/io/harness/beans/steps/nodes/RestoreCacheGCSNode.java:41`
- `877-pipeline-ci-cd-commons/src/java/io/harness/beans/steps/nodes/RestoreCacheGCSNode.java:51`

Plan creator:
- `332-ci-manager/service/src/main/java/io/harness/ci/execution/plancreator/RestoreCacheGCSStepPlanCreator.java:19`

Executor (engine step):
- `332-ci-manager/service/src/main/java/io/harness/ci/execution/registrars/ExecutionRegistrar.java:101`
- `332-ci-manager/service/src/main/java/io/harness/ci/execution/states/RestoreCacheGCSStep.java:16`

User-facing spec model (inputs live here):
- `879-pipeline-ci-commons/src/main/java/io/harness/beans/steps/stepinfo/RestoreCacheGCSStepInfo.java:52`

Default retry value:
- `879-pipeline-ci-commons/src/main/java/io/harness/beans/steps/stepinfo/RestoreCacheGCSStepInfo.java:59`

Step env assembly entry point for Restore Cache From GCS:
- `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:371`
- `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:1619`

Plugin images configured for Restore Cache From GCS:
- K8 plugin image: `332-ci-manager/config/ci-manager-config.yml:465`
- VM containerless plugin: `332-ci-manager/config/ci-manager-config.yml:676`

## Phase 2: GCP Connector Auth Methods & Required Fields

Supported GCP credential types:
- `954-connector-beans/src/main/java/io/harness/delegate/beans/connector/gcpconnector/GcpCredentialType.java:13`

Required fields by auth method:

Manual credentials:
- Required field: `secretKeyRef` (service account key).
- `954-connector-beans/src/main/java/io/harness/delegate/beans/connector/gcpconnector/GcpManualDetailsDTO.java:31`

OIDC:
- Required fields: `workloadPoolId`, `providerId`, `gcpProjectId`, `serviceAccountEmail`.
- `954-connector-beans/src/main/java/io/harness/delegate/beans/connector/gcpconnector/GcpOidcDetailsDTO.java:29`

Inherit from delegate:
- No additional fields on the connector spec (uses delegate credentials).
- `954-connector-beans/src/main/java/io/harness/delegate/beans/connector/gcpconnector/GcpCredentialType.java:14`

Connector detail assembly by auth type:
- `879-pipeline-ci-commons/src/main/java/io/harness/ci/utils/BaseConnectorUtils.java:645`

## Phase 3: Connector → Plugin Env Mapping (Per Auth Flow)

### Mapping Chain Overview

Connector env var names are decided by step type:
- Restore Cache GCS secret env name mapping: `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:712`

The mapping is applied on both K8 and VM paths:
- K8 conversion info uses the mapping: `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/integrationstage/k8s/K8InitializeStepUtils.java:2220`
- VM serializer merges the mapping into connector details: `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/serializer/vm/VmPluginCompatibleStepSerializer.java:267`

OIDC values are generated and injected through connector details:
- OIDC env map creation: `879-pipeline-ci-commons/src/main/java/io/harness/utils/GcpOidcAuthenticator.java:35`
- OIDC map attached to connector details: `879-pipeline-ci-commons/src/main/java/io/harness/ci/utils/BaseConnectorUtils.java:658`
- OIDC envs are pulled into step envs: `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:2458`
- OIDC envs are merged into Restore Cache GCS step envs early: `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:1622`

Manual credentials are materialized as secrets:
- Manual GCP secret mapping into env vars: `933-ci-commons/src/main/java/io/harness/connector/ConnectorEnvVariablesHelper.java:395`

### Auth Flow Details

```
Auth Method: Manual Credentials (Service Account Key)
├── Connector Fields Used: secretKeyRef
├── Mapped to Plugin Env/Args: PLUGIN_JSON_KEY
└── Code Location: env var names `PluginSettingUtils.java:712`; secret values `ConnectorEnvVariablesHelper.java:395`
```

```
Auth Method: OIDC
├── Connector Fields Used: workloadPoolId, providerId, gcpProjectId, serviceAccountEmail
├── Mapped to Plugin Env/Args: PLUGIN_PROJECT_NUMBER, PLUGIN_PROVIDER_ID, PLUGIN_POOL_ID, PLUGIN_SERVICE_ACCOUNT_EMAIL, PLUGIN_OIDC_TOKEN_ID
└── Code Location: env map creation `GcpOidcAuthenticator.java:99`; injected `PluginSettingUtils.java:2458`
```

```
Auth Method: Inherit From Delegate
├── Connector Fields Used: none (delegate credentials)
├── Mapped to Plugin Env/Args: None directly
└── Code Location: connector type `GcpCredentialType.java:14`
```

## Phase 4: Plugin Inputs Summary (All Inputs)

Primary Restore Cache GCS env assembly:
- `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:1619`

### Step Inputs → Plugin Env Vars

| Input Name | Type | Required | Description | Maps To (Plugin) |
|---|---|---:|---|---|
| connectorRef | string | Yes | GCP connector reference | Connector-derived envs (see auth flow matrix) |
| key | string | Yes | Cache key | `PLUGIN_CACHE_KEY` (`PluginSettingUtils.java:1639`) |
| bucket | string | Yes | GCS bucket | `PLUGIN_BUCKET` (`PluginSettingUtils.java:1643`) |
| sourcePaths | list<string> | No | Paths to restore | `PLUGIN_MOUNT` only for implicit step (`PluginSettingUtils.java:1631`) |
| preserveMetadata | boolean | No | Preserve file metadata | `PLUGIN_PRESERVE_METADATA` (`PluginSettingUtils.java:1653`) |
| failIfKeyNotFound | boolean | No | Fail if cache key is missing | `PLUGIN_FAIL_RESTORE_IF_KEY_NOT_PRESENT` (`PluginSettingUtils.java:1655`) |
| archiveFormat | enum | No | Cache archive format | `PLUGIN_ARCHIVE_FORMAT` (`PluginSettingUtils.java:1649`) |
| runAsUser | integer | No | Container user override | VM runtime setting, not plugin env (`VmPluginCompatibleStepSerializer.java:237`) |
| resources | object | No | Container resources | Container runtime setting (`RestoreCacheGCSStepInfo.java:77`) |
| retry | integer | No | Step retry count | Defaults to 1 (`RestoreCacheGCSStepInfo.java:59`) |

Additional plugin envs set by the builder:
- `PLUGIN_RESTORE=true`: `PluginSettingUtils.java:1646`
- `PLUGIN_EXIT_CODE=true`: `PluginSettingUtils.java:1647`
- `PLUGIN_BACKEND=gcs`: `PluginSettingUtils.java:1658`
- `PLUGIN_BACKEND_OPERATION_TIMEOUT`: `PluginSettingUtils.java:1659`
- `PLUGIN_CACHE_INTEL_METRICS_FILE`: `PluginSettingUtils.java:1660`

### Auth Flow Matrix

| Auth Method | Connector Fields | Plugin Env Vars | Notes |
|---|---|---|---|
| Manual Credentials | `secretKeyRef` | `PLUGIN_JSON_KEY` | Env names: `PluginSettingUtils.java:712`; values: `ConnectorEnvVariablesHelper.java:395` |
| OIDC | `workloadPoolId`, `providerId`, `gcpProjectId`, `serviceAccountEmail` | `PLUGIN_PROJECT_NUMBER`, `PLUGIN_PROVIDER_ID`, `PLUGIN_POOL_ID`, `PLUGIN_SERVICE_ACCOUNT_EMAIL`, `PLUGIN_OIDC_TOKEN_ID` | Created in `GcpOidcAuthenticator.java:99`; injected via `PluginSettingUtils.java:2458` |
| Inherit From Delegate | (none) | None directly | `GcpCredentialType.java:14` |

## Infra Applicability (K8 vs VM vs Hosted/DLITE)

Infra types for CI steps are:
- `K8`, `VM`, and `DLITE_VM`: `879-pipeline-ci-commons/src/main/java/io/harness/beans/sweepingoutputs/StageInfraDetails.java:33`

K8 flows call the plugin env builder with `Type.K8`:
- `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/serializer/PluginCompatibleStepSerializer.java:76`

VM and hosted/DLITE flows call it with `Type.VM`:
- `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/serializer/vm/VmPluginCompatibleStepSerializer.java:112`

The Restore Cache GCS env builder does not branch on infra type:
- `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:1619`

### Applicability Matrix

| Concern | K8 | VM | Hosted/DLITE_VM | Notes |
|---|---:|---:|---:|---|
| Connector auth env name mapping (GCP key → plugin envs) | Yes | Yes | Yes (via VM path) | Central mapping: `PluginSettingUtils.java:712`; K8 uses it: `K8InitializeStepUtils.java:2220`; VM merges it: `VmPluginCompatibleStepSerializer.java:267` |
| OIDC env injection (`PLUGIN_OIDC_TOKEN_ID`, `PLUGIN_SERVICE_ACCOUNT_EMAIL`, etc.) | Yes | Yes | Yes (via VM path) | OIDC injected before step envs: `PluginSettingUtils.java:1622`, `PluginSettingUtils.java:2458` |
| Core cache plugin envs (`PLUGIN_BUCKET`, `PLUGIN_CACHE_KEY`, `PLUGIN_RESTORE`, etc.) | Yes | Yes | Yes (via VM path) | Assembly in `getRestoreCacheGCSStepInfoEnvVariables`: `PluginSettingUtils.java:1619` |

## Key Takeaways

- Restore Cache From GCS uses the GCP connector and supports manual, OIDC, and inherit-from-delegate auth flows.
- Connector auth values are mapped to plugin envs (`PLUGIN_JSON_KEY` for manual; OIDC envs for workload identity).
- Restore behavior is configured via `PLUGIN_CACHE_KEY`, `PLUGIN_BUCKET`, `PLUGIN_RESTORE`, and related plugin flags in the env builder.
