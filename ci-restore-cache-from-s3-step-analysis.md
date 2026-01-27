# Restore Cache From S3: Inputs & AWS Connector Mapping

This document traces the CI Manager implementation of the **Restore Cache From S3** step, its user-facing inputs, AWS connector auth flows, connector → plugin env mapping, and infra-type applicability.

All code locations below are workspace-relative `path:line` references.

type: RestoreCacheS3
Plugin used: plugins/cache:1.9.17 (K8), github.com/drone-plugins/drone-meltwater-cache@refs/tags/v1.9.17 (VM containerless)

## Known Gaps / Gotchas

- `key` is `@NotNull` in the spec, but becomes optional when the step identifier is the implicit Harness cache step (`restore-cache-harness`).
  - Spec field: `879-pipeline-ci-commons/src/main/java/io/harness/beans/steps/stepinfo/RestoreCacheS3StepInfo.java:83`
  - Optional handling for `restore-cache-harness`: `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:1880`
  - Implicit step ID constant: `879-pipeline-ci-commons/src/main/java/io/harness/ci/commonconstants/CIExecutionConstants.java:37`
- `sourcePaths` is optional in the spec and only set into envs for the implicit Harness cache step.
  - Spec field: `879-pipeline-ci-commons/src/main/java/io/harness/beans/steps/stepinfo/RestoreCacheS3StepInfo.java:86`
  - Env set only when `restore-cache-harness`: `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:1884`
- `region` is optional in the spec, but the env builder resolves it with `required=true` before setting it as optional.
  - Spec field: `879-pipeline-ci-commons/src/main/java/io/harness/beans/steps/stepinfo/RestoreCacheS3StepInfo.java:90`
  - Builder resolution: `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:1908`
- AWS manual credentials include `sessionTokenRef`, but Restore Cache From S3 does not map it into plugin envs.
  - Spec: `954-connector-beans/src/main/java/io/harness/delegate/beans/connector/awsconnector/AwsManualConfigSpecDTO.java:35`
  - Secret mapping lacks session token: `933-ci-commons/src/main/java/io/harness/connector/ConnectorEnvVariablesHelper.java:314`
- Cross-account `externalId` is supported by the connector, but Restore Cache From S3 does not map `AWS_CROSS_ACCOUNT_EXTERNAL_ID`.
  - Cross-account DTO: `954-connector-beans/src/main/java/io/harness/delegate/beans/connector/awsconnector/CrossAccountAccessDTO.java:30`
  - Env name mapping for Restore Cache S3 lacks external ID: `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:698`
- There is no separate “Cloud” infra type; hosted/DLITE flows go through the VM serializer path.
  - Infra types: `879-pipeline-ci-commons/src/main/java/io/harness/beans/sweepingoutputs/StageInfraDetails.java:33`
  - VM serializer path: `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/serializer/vm/VmPluginCompatibleStepSerializer.java:112`

## Phase 1: Step Definition & Executor

Step registration and display metadata:
- `332-ci-manager/service/src/main/java/io/harness/ci/execution/plan/creator/CIPipelineServiceInfoProvider.java:294`

Step node wiring to the spec:
- `877-pipeline-ci-cd-commons/src/java/io/harness/beans/steps/nodes/RestoreCacheS3Node.java:41`
- `877-pipeline-ci-cd-commons/src/java/io/harness/beans/steps/nodes/RestoreCacheS3Node.java:51`

Plan creator:
- `332-ci-manager/service/src/main/java/io/harness/ci/execution/plancreator/RestoreCacheS3StepPlanCreator.java:19`

Executor (engine step):
- `332-ci-manager/service/src/main/java/io/harness/ci/execution/registrars/ExecutionRegistrar.java:98`
- `332-ci-manager/service/src/main/java/io/harness/ci/execution/states/RestoreCacheS3Step.java:16`

User-facing spec model (inputs live here):
- `879-pipeline-ci-commons/src/main/java/io/harness/beans/steps/stepinfo/RestoreCacheS3StepInfo.java:52`

Default retry value:
- `879-pipeline-ci-commons/src/main/java/io/harness/beans/steps/stepinfo/RestoreCacheS3StepInfo.java:63`

Step env assembly entry point for Restore Cache From S3:
- `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:384`
- `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:1876`

Plugin images configured for Restore Cache From S3:
- K8 plugin image: `332-ci-manager/config/ci-manager-config.yml:477`
- VM containerless plugin: `332-ci-manager/config/ci-manager-config.yml:672`

## Phase 2: AWS Connector Auth Methods & Required Fields

Supported AWS credential types:
- `954-connector-beans/src/main/java/io/harness/delegate/beans/connector/awsconnector/AwsCredentialType.java:18`

Required fields by auth method:

Manual credentials:
- Required fields: `accessKey` or `accessKeyRef`, and `secretKeyRef`.
- `954-connector-beans/src/main/java/io/harness/delegate/beans/connector/awsconnector/AwsManualConfigSpecDTO.java:32`
- Optional field: `sessionTokenRef`.
- `954-connector-beans/src/main/java/io/harness/delegate/beans/connector/awsconnector/AwsManualConfigSpecDTO.java:35`

OIDC:
- Required field: `iamRoleArn`.
- `954-connector-beans/src/main/java/io/harness/delegate/beans/connector/awsconnector/AwsOidcSpecDTO.java:29`

Inherit from delegate:
- Requires `delegateSelectors` on the connector for `INHERIT_FROM_DELEGATE`.
- `954-connector-beans/src/main/java/io/harness/delegate/beans/connector/AwsConnectorDTO.java:90`

IRSA:
- Requires `delegateSelectors` on the connector for `IRSA`.
- `954-connector-beans/src/main/java/io/harness/delegate/beans/connector/AwsConnectorDTO.java:90`

Cross-account access:
- `crossAccountRoleArn` is required if cross-account access is used; `externalId` is optional.
- `954-connector-beans/src/main/java/io/harness/delegate/beans/connector/awsconnector/CrossAccountAccessDTO.java:30`

Connector detail assembly by auth type:
- `879-pipeline-ci-commons/src/main/java/io/harness/ci/utils/BaseConnectorUtils.java:568`

## Phase 3: Connector → Plugin Env Mapping (Per Auth Flow)

### Mapping Chain Overview

Connector env var names are decided by step type:
- Restore Cache S3 secret env name mapping: `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:698`

The mapping is applied on both K8 and VM paths:
- K8 conversion info uses the mapping: `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/integrationstage/k8s/K8InitializeStepUtils.java:2220`
- VM serializer merges the mapping into connector details: `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/serializer/vm/VmPluginCompatibleStepSerializer.java:267`

OIDC values are generated and injected through connector details:
- OIDC env map creation: `879-pipeline-ci-commons/src/main/java/io/harness/utils/AwsOidcAuthenticator.java:57`
- OIDC map attached to connector details: `879-pipeline-ci-commons/src/main/java/io/harness/ci/utils/BaseConnectorUtils.java:581`
- OIDC envs are pulled into step envs: `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:2458`
- OIDC envs are merged into Restore Cache S3 step envs early: `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:1895`

Manual credentials are materialized as secrets:
- Manual AWS secret mapping into env vars: `933-ci-commons/src/main/java/io/harness/connector/ConnectorEnvVariablesHelper.java:314`

### Auth Flow Details

```
Auth Method: Manual Credentials (Access Key + Secret Key)
├── Connector Fields Used: accessKey or accessKeyRef, secretKeyRef
├── Mapped to Plugin Env/Args: PLUGIN_ACCESS_KEY, PLUGIN_SECRET_KEY
└── Code Location: env var names `PluginSettingUtils.java:698`; secret values `ConnectorEnvVariablesHelper.java:314`
```

```
Auth Method: Manual Credentials (Cross-Account Role)
├── Connector Fields Used: crossAccountRoleArn, optional externalId
├── Mapped to Plugin Env/Args: PLUGIN_ASSUME_ROLE (externalId not mapped for Restore Cache S3)
└── Code Location: env var names `PluginSettingUtils.java:698`; cross-account values `ConnectorEnvVariablesHelper.java:352`
```

```
Auth Method: OIDC
├── Connector Fields Used: iamRoleArn
├── Mapped to Plugin Env/Args: PLUGIN_ASSUME_ROLE, PLUGIN_OIDC_TOKEN_ID
└── Code Location: env map creation `AwsOidcAuthenticator.java:57`; injected `PluginSettingUtils.java:2458`
```

```
Auth Method: Inherit From Delegate
├── Connector Fields Used: delegateSelectors
├── Mapped to Plugin Env/Args: None directly
└── Code Location: selectors required `AwsConnectorDTO.java:90`
```

```
Auth Method: IRSA
├── Connector Fields Used: delegateSelectors
├── Mapped to Plugin Env/Args: None directly
└── Code Location: selectors required `AwsConnectorDTO.java:90`
```

## Phase 4: Plugin Inputs Summary (All Inputs)

Primary Restore Cache S3 env assembly:
- `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:1876`

### Step Inputs → Plugin Env Vars

| Input Name | Type | Required | Description | Maps To (Plugin) |
|---|---|---:|---|---|
| connectorRef | string | Yes | AWS connector reference | Connector-derived envs (see auth flow matrix) |
| key | string | Yes | Cache key | `PLUGIN_CACHE_KEY` (`PluginSettingUtils.java:1892`) |
| bucket | string | Yes | S3 bucket | `PLUGIN_BUCKET` (`PluginSettingUtils.java:1899`) |
| sourcePaths | list<string> | No | Paths to restore | `PLUGIN_MOUNT` only for implicit step (`PluginSettingUtils.java:1884`) |
| region | string | No | AWS region | `PLUGIN_REGION` (`PluginSettingUtils.java:1908`) |
| endpoint | string | No | S3 endpoint override | `PLUGIN_ENDPOINT` (`PluginSettingUtils.java:1903`) |
| pathStyle | boolean | No | Force path-style addressing | `PLUGIN_PATH_STYLE` (`PluginSettingUtils.java:1917`) |
| preserveMetadata | boolean | No | Preserve file metadata | `PLUGIN_PRESERVE_METADATA` (`PluginSettingUtils.java:1920`) |
| failIfKeyNotFound | boolean | No | Fail if cache key is missing | `PLUGIN_FAIL_RESTORE_IF_KEY_NOT_PRESENT` (`PluginSettingUtils.java:1922`) |
| archiveFormat | enum | No | Cache archive format | `PLUGIN_ARCHIVE_FORMAT` (`PluginSettingUtils.java:1913`) |
| runAsUser | integer | No | Container user override | VM runtime setting, not plugin env (`VmPluginCompatibleStepSerializer.java:237`) |
| resources | object | No | Container resources | Container runtime setting (`RestoreCacheS3StepInfo.java:79`) |
| retry | integer | No | Step retry count | Defaults to 1 (`RestoreCacheS3StepInfo.java:63`) |

Additional plugin envs set by the builder:
- `PLUGIN_RESTORE=true`: `PluginSettingUtils.java:1901`
- `PLUGIN_EXIT_CODE=true`: `PluginSettingUtils.java:1925`
- `PLUGIN_BACKEND=s3`: `PluginSettingUtils.java:1926`
- `PLUGIN_BACKEND_OPERATION_TIMEOUT`: `PluginSettingUtils.java:1927`
- `PLUGIN_CACHE_INTEL_METRICS_FILE`: `PluginSettingUtils.java:1928`

### Auth Flow Matrix

| Auth Method | Connector Fields | Plugin Env Vars | Notes |
|---|---|---|---|
| Manual Credentials | `accessKey|accessKeyRef`, `secretKeyRef` | `PLUGIN_ACCESS_KEY`, `PLUGIN_SECRET_KEY` | Env names: `PluginSettingUtils.java:698`; values: `ConnectorEnvVariablesHelper.java:314` |
| Manual + Cross-Account | `crossAccountRoleArn`, optional `externalId` | `PLUGIN_ASSUME_ROLE` | External ID not mapped for Restore Cache S3: `PluginSettingUtils.java:698` |
| OIDC | `iamRoleArn` | `PLUGIN_ASSUME_ROLE`, `PLUGIN_OIDC_TOKEN_ID` | Created in `AwsOidcAuthenticator.java:57`; injected via `PluginSettingUtils.java:2458` |
| Inherit From Delegate | `delegateSelectors` | None directly | Selectors required: `AwsConnectorDTO.java:90` |
| IRSA | `delegateSelectors` | None directly | Selectors required: `AwsConnectorDTO.java:90` |

## Infra Applicability (K8 vs VM vs Hosted/DLITE)

Infra types for CI steps are:
- `K8`, `VM`, and `DLITE_VM`: `879-pipeline-ci-commons/src/main/java/io/harness/beans/sweepingoutputs/StageInfraDetails.java:33`

K8 flows call the plugin env builder with `Type.K8`:
- `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/serializer/PluginCompatibleStepSerializer.java:76`

VM and hosted/DLITE flows call it with `Type.VM`:
- `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/serializer/vm/VmPluginCompatibleStepSerializer.java:112`

The Restore Cache S3 env builder does not branch on infra type:
- `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:1876`

### Applicability Matrix

| Concern | K8 | VM | Hosted/DLITE_VM | Notes |
|---|---:|---:|---:|---|
| Connector auth env name mapping (AWS keys → plugin envs) | Yes | Yes | Yes (via VM path) | Central mapping: `PluginSettingUtils.java:698`; K8 uses it: `K8InitializeStepUtils.java:2220`; VM merges it: `VmPluginCompatibleStepSerializer.java:267` |
| OIDC env injection (`PLUGIN_ASSUME_ROLE`, `PLUGIN_OIDC_TOKEN_ID`) | Yes | Yes | Yes (via VM path) | OIDC injected before step envs: `PluginSettingUtils.java:1895`, `PluginSettingUtils.java:2458` |
| Core cache plugin envs (`PLUGIN_BUCKET`, `PLUGIN_CACHE_KEY`, `PLUGIN_RESTORE`, etc.) | Yes | Yes | Yes (via VM path) | Assembly in `getRestoreCacheS3StepInfoEnvVariables`: `PluginSettingUtils.java:1876` |

## Key Takeaways

- Restore Cache From S3 uses the AWS connector and supports manual, OIDC, inherit-from-delegate, and IRSA auth flows.
- Connector auth values are mapped to plugin-specific env vars (`PLUGIN_ACCESS_KEY`, `PLUGIN_SECRET_KEY`, `PLUGIN_ASSUME_ROLE`).
- Restore behavior is configured via `PLUGIN_CACHE_KEY`, `PLUGIN_BUCKET`, `PLUGIN_RESTORE`, and related plugin flags in the env builder.
