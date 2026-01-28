# Upload to S3: Inputs & AWS Connector Mapping

This document traces the CI Manager implementation of the **Upload Artifacts to S3** step, its user-facing inputs, AWS connector auth flows, connector → plugin env mapping, and infra-type applicability.

All code locations below are workspace-relative `path:line` references.

type: S3Upload
Plugin used: plugins/s3:1.5.5 (K8), github.com/drone-plugins/drone-s3@refs/tags/v1.5.5 (VM containerless)

## Known Gaps / Gotchas

- The Upload to S3 env builder does not set `PLUGIN_ENABLE_PROXY`, even if the AWS connector uses a proxy.
- `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:2153`
- AWS manual credentials include `sessionTokenRef`, but Upload to S3 does not map it into plugin envs.
- `954-connector-beans/src/main/java/io/harness/delegate/beans/connector/awsconnector/AwsManualConfigSpecDTO.java:32`
- `933-ci-commons/src/main/java/io/harness/connector/ConnectorEnvVariablesHelper.java:314`
- Cross-account `externalId` is supported on the connector, but Upload to S3 does not map `AWS_CROSS_ACCOUNT_EXTERNAL_ID`.
- `954-connector-beans/src/main/java/io/harness/delegate/beans/connector/awsconnector/CrossAccountAccessDTO.java:30`
- `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:698`
- There is no separate “Cloud” infra type; hosted/DLITE flows go through the VM serializer path.
- Infra types: `879-pipeline-ci-commons/src/main/java/io/harness/beans/sweepingoutputs/StageInfraDetails.java:33`
- VM serializer path: `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/serializer/vm/VmPluginCompatibleStepSerializer.java:90`

## Phase 1: Step Definition & Executor

Step registration and display metadata:
- `332-ci-manager/service/src/main/java/io/harness/ci/execution/plan/creator/CIPipelineServiceInfoProvider.java:387`

Step node wiring to the spec:
- `332-ci-manager/service/src/main/java/io/harness/ci/beans/steps/nodes/S3UploadNode.java:35`
- `332-ci-manager/service/src/main/java/io/harness/ci/beans/steps/nodes/S3UploadNode.java:45`

Plan creator:
- `332-ci-manager/service/src/main/java/io/harness/ci/execution/plancreator/S3UploadStepPlanCreator.java:19`

Executor (engine step):
- `332-ci-manager/service/src/main/java/io/harness/ci/execution/registrars/ExecutionRegistrar.java:96`
- `332-ci-manager/service/src/main/java/io/harness/ci/execution/states/UploadToS3Step.java:25`

User-facing spec model (inputs live here):
- `879-pipeline-ci-commons/src/main/java/io/harness/beans/steps/stepinfo/UploadToS3StepInfo.java:53`

Default retry value:
- `879-pipeline-ci-commons/src/main/java/io/harness/beans/steps/stepinfo/UploadToS3StepInfo.java:59`

Step env assembly entry point for S3 upload:
- `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:348`
- `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:2153`

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
- S3 upload secret env name mapping: `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:698`

The mapping is applied on both K8 and VM paths:
- K8 conversion info uses the mapping: `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/integrationstage/k8s/K8InitializeStepUtils.java:2220`
- VM serializer merges the mapping into connector details: `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/serializer/vm/VmPluginCompatibleStepSerializer.java:267`

OIDC values are generated and injected through connector details:
- OIDC env map creation: `879-pipeline-ci-commons/src/main/java/io/harness/utils/AwsOidcAuthenticator.java:57`
- OIDC map attached to connector details: `879-pipeline-ci-commons/src/main/java/io/harness/ci/utils/BaseConnectorUtils.java:581`
- OIDC envs are pulled into step envs: `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:2458`
- OIDC envs are merged into S3 upload step envs early: `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:2156`

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
├── Mapped to Plugin Env/Args: PLUGIN_ASSUME_ROLE (externalId not mapped for Upload to S3)
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
├── Mapped to Plugin Env/Args: none directly (delegate identity is used)
└── Code Location: validation `AwsConnectorDTO.java:90`
```

```
Auth Method: IRSA
├── Connector Fields Used: delegateSelectors
├── Mapped to Plugin Env/Args: none directly (delegate identity is used)
└── Code Location: validation `AwsConnectorDTO.java:90`
```

## Phase 4: Plugin Inputs Summary (All Inputs)

Primary S3 upload env assembly:
- `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:2153`

Plugin images configured for S3 upload:
- `332-ci-manager/config/ci-manager-config.yml:441`

### Step Inputs → Plugin Env Vars

| Input Name | Type | Required | Description | Maps To (Plugin) |
|---|---|---:|---|---|
| connectorRef | string | Yes | AWS connector reference | Connector-derived envs (see auth flow matrix); mapping defined at `PluginSettingUtils.java:698` |
| bucket | string | Yes | S3 bucket name | `PLUGIN_BUCKET` (`PluginSettingUtils.java:2161`) |
| sourcePath | string | Yes | Local path to upload | `PLUGIN_SOURCE` (`PluginSettingUtils.java:2163`) |
| target | string | No | Object key prefix within bucket | `PLUGIN_TARGET` (`PluginSettingUtils.java:2166`) |
| endpoint | string | No | Custom S3 endpoint | `PLUGIN_ENDPOINT` (`PluginSettingUtils.java:2171`) |
| region | string | Yes | AWS region | `PLUGIN_REGION` (`PluginSettingUtils.java:2176`) |
| stripPrefix | string | No | Prefix to strip from source | `PLUGIN_STRIP_PREFIX` (`PluginSettingUtils.java:2181`) |
| runAsUser | integer | No | Container user override | Runtime container setting (`VmPluginCompatibleStepSerializer.java:237`) |
| resources | object | No | Container resources | Runtime container setting (`UploadToS3StepInfo.java:73`) |
| retry | integer | No | Step retry count | Defaults to 1 (`UploadToS3StepInfo.java:59`) |

### Auth Flow Matrix

| Auth Method | Connector Fields | Plugin Env Vars | Notes |
|---|---|---|---|
| Manual Credentials | `accessKey` or `accessKeyRef`, `secretKeyRef` | `PLUGIN_ACCESS_KEY`, `PLUGIN_SECRET_KEY` | Env names from `PluginSettingUtils.java:698`; values from `ConnectorEnvVariablesHelper.java:314` |
| Manual Credentials + Cross-Account | `crossAccountRoleArn`, optional `externalId` | `PLUGIN_ASSUME_ROLE` | External ID is not mapped for Upload to S3: `PluginSettingUtils.java:698` |
| OIDC | `iamRoleArn` | `PLUGIN_ASSUME_ROLE`, `PLUGIN_OIDC_TOKEN_ID` | Created in `AwsOidcAuthenticator.java:57`; injected via `PluginSettingUtils.java:2458` |
| Inherit From Delegate | `delegateSelectors` | None directly | Selectors required: `AwsConnectorDTO.java:90` |
| IRSA | `delegateSelectors` | None directly | Selectors required: `AwsConnectorDTO.java:90` |

## Infra Applicability (K8 vs VM vs Hosted/DLITE)

Infra types for CI steps are:
- `K8`, `VM`, and `DLITE_VM`: `879-pipeline-ci-commons/src/main/java/io/harness/beans/sweepingoutputs/StageInfraDetails.java:33`

K8 flows call the plugin env builder with `Type.K8`:
- `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/serializer/PluginCompatibleStepSerializer.java:68`

VM and hosted/DLITE flows use the VM serializer path:
- `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/serializer/vm/VmPluginCompatibleStepSerializer.java:90`

The S3 upload step uses the same env builder for all infra types:
- `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:2153`

### Applicability Matrix

| Concern | K8 | VM | Hosted/DLITE_VM | Notes |
|---|---:|---:|---:|---|
| Connector auth env name mapping (AWS keys → plugin envs) | Yes | Yes | Yes (via VM path) | Central mapping: `PluginSettingUtils.java:698`; K8 uses it: `K8InitializeStepUtils.java:2220`; VM merges it: `VmPluginCompatibleStepSerializer.java:267` |
| OIDC env injection (`PLUGIN_ASSUME_ROLE`, `PLUGIN_OIDC_TOKEN_ID`) | Yes | Yes | Yes (via VM path) | OIDC injected before step envs: `PluginSettingUtils.java:2156`, `PluginSettingUtils.java:2458` |
| Core upload plugin envs (`PLUGIN_BUCKET`, `PLUGIN_SOURCE`, `PLUGIN_TARGET`, etc.) | Yes | Yes | Yes (via VM path) | Assembly in `getUploadToS3StepInfoEnvVariables`: `PluginSettingUtils.java:2153` |

## Key Takeaways

- Upload to S3 uses the AWS connector with manual, OIDC, inherit-from-delegate, and IRSA auth flows.
- S3 upload maps bucket, source, region, and optional endpoint/target/stripPrefix into the plugin envs.
- Cross-account `externalId` and session tokens are not mapped for this step.
