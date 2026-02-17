# Build and Push to ECR: Inputs & AWS Connector Mapping

This document traces the CI Manager implementation of the **Build and Push to ECR** step, its user-facing inputs, AWS connector auth flows, connector → plugin env mapping, and infra-type applicability.

All code locations below are workspace-relative `path:line` references.

type: BuildAndPushECR
Plugin used: plugins/kaniko-ecr:1.13.3 (K8), plugins/buildx-ecr:1.4.2 (buildx), plugins/ecr:21.1.1 (VM)

## Known Gaps / Gotchas

- AWS session token is defined on the manual AWS connector spec, but this step’s connector secret mapping does not populate a session-token env var for ECR builds.
  - Spec includes `sessionTokenRef`: `954-connector-beans/src/main/java/io/harness/delegate/beans/connector/awsconnector/AwsManualConfigSpecDTO.java:35`
  - Manual creds + cross-account values are mapped here (no session token): `933-ci-commons/src/main/java/io/harness/connector/ConnectorEnvVariablesHelper.java:323`
- There is no separate “Cloud” infra type at this layer. Hosted/DLITE runs through the VM path for env mapping.
  - Infra types: `K8`, `VM`, `DLITE_VM`: `879-pipeline-ci-commons/src/main/java/io/harness/beans/sweepingoutputs/StageInfraDetails.java:33`
  - VM/hosted uses `Type.VM` when building plugin envs: `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/serializer/vm/VmPluginCompatibleStepSerializer.java:112`
  - K8 uses `Type.K8`: `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/serializer/PluginCompatibleStepSerializer.java:76`
- AWS connector auth values are mapped to plugin-specific `PLUGIN_*` env vars (not standard `AWS_*` names) for this step.
  - ECR connector env var names: `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:692`

## Phase 1: Step Definition & Executor

Step registration and display metadata:
- `332-ci-manager/service/src/main/java/io/harness/ci/execution/plan/creator/CIPipelineServiceInfoProvider.java:340`

Step node wiring to the spec:
- `332-ci-manager/service/src/main/java/io/harness/ci/beans/steps/nodes/BuildAndPushECRNode.java:35`
- `332-ci-manager/service/src/main/java/io/harness/ci/beans/steps/nodes/BuildAndPushECRNode.java:45`

Plan creator:
- `332-ci-manager/service/src/main/java/io/harness/ci/execution/plancreator/BuildAndPushECRStepPlanCreator.java:19`

Executor (engine step):
- `332-ci-manager/service/src/main/java/io/harness/ci/execution/registrars/ExecutionRegistrar.java:91`
- `332-ci-manager/service/src/main/java/io/harness/ci/execution/states/ECRStep.java:50`

User-facing spec model (inputs live here):
- `879-pipeline-ci-commons/src/main/java/io/harness/beans/steps/stepinfo/ECRStepInfo.java:83`

Default retry value:
- `879-pipeline-ci-commons/src/main/java/io/harness/beans/steps/stepinfo/ECRStepInfo.java:59`

## Phase 2: AWS Connector Auth Methods & Required Fields

Supported AWS credential types:
- `954-connector-beans/src/main/java/io/harness/delegate/beans/connector/awsconnector/AwsCredentialType.java:18`

Required fields by auth method:
- Access Key + Secret Key (manual credentials):
  - Access key: one of `accessKey` or `accessKeyRef`: `954-connector-beans/src/main/java/io/harness/delegate/beans/connector/awsconnector/AwsManualConfigSpecDTO.java:28`
  - Secret key ref is required: `954-connector-beans/src/main/java/io/harness/delegate/beans/connector/awsconnector/AwsManualConfigSpecDTO.java:34`
- OIDC / Web Identity:
  - IAM role ARN is required: `954-connector-beans/src/main/java/io/harness/delegate/beans/connector/awsconnector/AwsOidcSpecDTO.java:29`
- IAM Role via Delegate (inherit from delegate):
  - Delegate selectors required: `954-connector-beans/src/main/java/io/harness/delegate/beans/connector/awsconnector/AwsInheritFromDelegateSpecDTO.java:27`
  - Validation enforces delegate selectors for inherit/IRSA: `954-connector-beans/src/main/java/io/harness/delegate/beans/connector/AwsConnectorDTO.java:91`
- IRSA:
  - No fields on the IRSA spec itself: `954-connector-beans/src/main/java/io/harness/delegate/beans/connector/awsconnector/AwsIRSASpecDTO.java:25`
  - Delegate selectors still enforced via validation: `954-connector-beans/src/main/java/io/harness/delegate/beans/connector/AwsConnectorDTO.java:92`
- Assume Role (cross-account overlay):
  - Cross-account role ARN is required when cross-account is configured: `954-connector-beans/src/main/java/io/harness/delegate/beans/connector/awsconnector/CrossAccountAccessDTO.java:30`

Notable gap for this step:
- `sessionTokenRef` exists on the AWS manual spec: `954-connector-beans/src/main/java/io/harness/delegate/beans/connector/awsconnector/AwsManualConfigSpecDTO.java:35`
- The CI connector env helper maps access key, secret key, and cross-account values, but not session token here: `933-ci-commons/src/main/java/io/harness/connector/ConnectorEnvVariablesHelper.java:323`

## Phase 3: Connector → Plugin Env Mapping (Per Auth Flow)

### Mapping Chain Overview

Connector env var names are decided by step type:
- ECR secret env name mapping: `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:692`

The mapping is applied on both K8 and VM paths:
- K8 conversion info uses the mapping: `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/integrationstage/k8s/K8InitializeStepUtils.java:2220`
- VM serializer merges the mapping into connector details: `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/serializer/vm/VmPluginCompatibleStepSerializer.java:267`

OIDC values are generated and injected through connector details:
- AWS OIDC env map population: `879-pipeline-ci-commons/src/main/java/io/harness/utils/AwsOidcAuthenticator.java:59`
- OIDC map attached to connector details: `879-pipeline-ci-commons/src/main/java/io/harness/ci/utils/BaseConnectorUtils.java:581`
- OIDC envs are pulled into step envs: `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:2458`
- ECR step env assembly merges OIDC envs first: `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:1201`

Manual credentials and cross-account values are materialized as secrets:
- Manual access/secret key mapping into env vars: `933-ci-commons/src/main/java/io/harness/connector/ConnectorEnvVariablesHelper.java:323`
- Cross-account role/external ID mapping into env vars: `933-ci-commons/src/main/java/io/harness/connector/ConnectorEnvVariablesHelper.java:353`

### Auth Flow Details

Auth Method: Access Key + Secret Key (Manual Credentials)
- Connector Fields Used:
  - `credential.spec.accessKey | accessKeyRef`
  - `credential.spec.secretKeyRef`
  - Optional overlay: `credential.crossAccountAccess.crossAccountRoleArn`, `externalId`
- Mapped to Plugin Env/Args:
  - `PLUGIN_ACCESS_KEY`, `PLUGIN_SECRET_KEY`
  - Optional overlay: `PLUGIN_ASSUME_ROLE`, `PLUGIN_EXTERNAL_ID`
- Code Location:
  - Env var name mapping: `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:692`
  - Manual values populated: `933-ci-commons/src/main/java/io/harness/connector/ConnectorEnvVariablesHelper.java:323`
  - Cross-account values populated: `933-ci-commons/src/main/java/io/harness/connector/ConnectorEnvVariablesHelper.java:353`

Auth Method: OIDC / Web Identity
- Connector Fields Used:
  - `credential.spec.iamRoleArn`
- Mapped to Plugin Env/Args:
  - `PLUGIN_ASSUME_ROLE`, `PLUGIN_OIDC_TOKEN_ID`
- Code Location:
  - OIDC env map creation: `879-pipeline-ci-commons/src/main/java/io/harness/utils/AwsOidcAuthenticator.java:59`
  - OIDC connector details population: `879-pipeline-ci-commons/src/main/java/io/harness/ci/utils/BaseConnectorUtils.java:581`
  - OIDC env injection into step envs: `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:2458`
  - OIDC envs merged into ECR step envs: `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:1201`

Auth Method: IAM Role via Delegate (Inherit From Delegate)
- Connector Fields Used:
  - `credential.spec.delegateSelectors`
- Mapped to Plugin Env/Args:
  - None directly. The delegate’s IAM context is used.
- Code Location:
  - Required selectors on spec: `954-connector-beans/src/main/java/io/harness/delegate/beans/connector/awsconnector/AwsInheritFromDelegateSpecDTO.java:27`
  - Validation enforcement: `954-connector-beans/src/main/java/io/harness/delegate/beans/connector/AwsConnectorDTO.java:91`

Auth Method: IRSA (EKS-specific)
- Connector Fields Used:
  - IRSA spec has no fields; delegate selectors are still enforced.
- Mapped to Plugin Env/Args:
  - None directly. IRSA on the delegate/EKS is used.
- Code Location:
  - IRSA spec: `954-connector-beans/src/main/java/io/harness/delegate/beans/connector/awsconnector/AwsIRSASpecDTO.java:25`
  - Validation enforcement: `954-connector-beans/src/main/java/io/harness/delegate/beans/connector/AwsConnectorDTO.java:92`

Auth Method: Assume Role (Cross-account overlay)
- Connector Fields Used:
  - `credential.crossAccountAccess.crossAccountRoleArn`
  - `credential.crossAccountAccess.externalId`
- Mapped to Plugin Env/Args:
  - `PLUGIN_ASSUME_ROLE`, `PLUGIN_EXTERNAL_ID`
- Code Location:
  - Cross-account DTO: `954-connector-beans/src/main/java/io/harness/delegate/beans/connector/awsconnector/CrossAccountAccessDTO.java:30`
  - Env var name mapping: `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:695`
  - Values populated: `933-ci-commons/src/main/java/io/harness/connector/ConnectorEnvVariablesHelper.java:353`

## Phase 4: Plugin Inputs Summary (All Inputs)

The primary plugin env assembly for this step is here:
- `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:1198`

Plugin images configured for ECR:
- Kaniko ECR: `332-ci-manager/config/ci-manager-config.yml:384`
- Buildx ECR: `332-ci-manager/config/ci-manager-config.yml:407`

### Step Inputs → Plugin Env Vars

| Input Name | Type | Required | Description | Maps To (Plugin) |
|---|---|---:|---|---|
| connectorRef | string | Yes | AWS connector reference | Connector-derived envs (see auth flow matrix) |
| account | string | Yes | AWS account ID | `PLUGIN_REGISTRY` (`PluginSettingUtils.java:1205`, `PluginSettingUtils.java:1212`) |
| region | string | Yes | AWS region | `PLUGIN_REGION` (`PluginSettingUtils.java:1220`) and registry (`PluginSettingUtils.java:1209`) |
| imageName | string | Yes | ECR repository/image name | `PLUGIN_REPO` (`PluginSettingUtils.java:1214`) |
| tags | list<string> | Yes | Image tags | `PLUGIN_TAGS` (`PluginSettingUtils.java:1216`) |
| dockerfile | string | No | Dockerfile path | `PLUGIN_DOCKERFILE` (`PluginSettingUtils.java:1223`) |
| context | string | No | Build context | `PLUGIN_CONTEXT` (`PluginSettingUtils.java:1229`) |
| target | string | No | Docker target stage | `PLUGIN_TARGET` (`PluginSettingUtils.java:1234`) |
| buildArgs | map<string,string> | No | Docker build args | `PLUGIN_BUILD_ARGS`, `PLUGIN_BUILD_ARGS_NEW` (`PluginSettingUtils.java:1240`) |
| labels | map<string,string> | No | Image labels | `PLUGIN_CUSTOM_LABELS` (`PluginSettingUtils.java:1246`) |
| optimize | boolean | No | K8 snapshot optimization (defaults true) | `PLUGIN_SNAPSHOT_MODE=redo` on K8 (`PluginSettingUtils.java:1286`) |
| caching | boolean | No | Enables buildx/DLC behavior | Buildx driver opts + cache metrics (`PluginSettingUtils.java:1261`) |
| cacheFrom | list<string> | No | Cache sources | `PLUGIN_CACHE_FROM` (`PluginSettingUtils.java:1307`) |
| cacheTo | string | No | Cache destination | `PLUGIN_CACHE_TO` (`PluginSettingUtils.java:1257`) |
| remoteCacheImage | string | No | Remote cache image | `PLUGIN_ENABLE_CACHE` + `PLUGIN_CACHE_REPO` (non-buildx) or appended to `cacheFrom` (`PluginSettingUtils.java:1293`) |
| baseImageConnectorRefs | list<string> | No | Base image connector(s) | Base image registry envs (`PluginSettingUtils.java:1278`) |
| envDockerSecrets | map<string,string> | No | Docker secrets via env (buildx) | `PLUGIN_SECRETS_FROM_ENV` (`PluginSettingUtils.java:1253`) |
| fileDockerSecrets | map<string,string> | No | Docker secrets via file (buildx) | `PLUGIN_SECRETS_FROM_FILE` (`PluginSettingUtils.java:1253`) |
| envVariables | map<string,string> | No | Extra env vars | Merged directly (`PluginSettingUtils.java:1321`) |
| runAsUser | integer | No | Container user override | Container runtime setting, not plugin env (`VmPluginCompatibleStepSerializer.java:112`) |
| resources | object | No | Container resources | Container runtime setting (`ECRStepInfo.java:84`) |
| retry | integer | No | Step retry count | Defaults to 1 (`ECRStepInfo.java:59`) |

### Auth Flow Matrix

| Auth Method | Connector Fields | Plugin Env Vars | Notes |
|---|---|---|---|
| Access Key + Secret Key | `accessKey|accessKeyRef`, `secretKeyRef` | `PLUGIN_ACCESS_KEY`, `PLUGIN_SECRET_KEY` | Env names: `PluginSettingUtils.java:692`; values: `ConnectorEnvVariablesHelper.java:323` |
| OIDC / Web Identity | `iamRoleArn` | `PLUGIN_ASSUME_ROLE`, `PLUGIN_OIDC_TOKEN_ID` | Created: `AwsOidcAuthenticator.java:59`; injected: `PluginSettingUtils.java:2458` |
| Inherit From Delegate | `delegateSelectors` | None directly | Selectors required: `AwsInheritFromDelegateSpecDTO.java:27`; enforced: `AwsConnectorDTO.java:91` |
| IRSA | (no spec fields), `delegateSelectors` | None directly | IRSA spec: `AwsIRSASpecDTO.java:25`; selectors enforced: `AwsConnectorDTO.java:92` |
| Assume Role (cross-account overlay) | `crossAccountRoleArn`, `externalId` | `PLUGIN_ASSUME_ROLE`, `PLUGIN_EXTERNAL_ID` | Env names: `PluginSettingUtils.java:695`; values: `ConnectorEnvVariablesHelper.java:353` |

## Infra Applicability (K8 vs VM vs Hosted/DLITE)

Infra types for CI steps are:
- `K8`, `VM`, and `DLITE_VM`: `879-pipeline-ci-commons/src/main/java/io/harness/beans/sweepingoutputs/StageInfraDetails.java:33`

K8 flows call the plugin env builder with `Type.K8`:
- `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/serializer/PluginCompatibleStepSerializer.java:76`

VM and hosted/DLITE flows call it with `Type.VM`:
- `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/serializer/vm/VmPluginCompatibleStepSerializer.java:112`

The ECR step’s env assembly explicitly branches on infra type:
- `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:1275`

### Applicability Matrix

| Concern | K8 | VM | Hosted/DLITE_VM | Notes |
|---|---:|---:|---:|---|
| Connector auth env name mapping (`AWS_*` → `PLUGIN_*`) | Yes | Yes | Yes (via VM path) | Central mapping: `PluginSettingUtils.java:692`; K8 uses it: `K8InitializeStepUtils.java:2220`; VM merges it: `VmPluginCompatibleStepSerializer.java:267` |
| OIDC env injection (`PLUGIN_ASSUME_ROLE`, `PLUGIN_OIDC_TOKEN_ID`) | Yes | Yes | Yes (via VM path) | OIDC injected before step-specific envs: `PluginSettingUtils.java:1201`, `PluginSettingUtils.java:2458` |
| Core ECR plugin envs (`PLUGIN_REGISTRY`, `PLUGIN_REPO`, `PLUGIN_TAGS`, etc.) | Yes | Yes | Yes (via VM path) | Common assembly in `getECRStepInfoEnvVariables`: `PluginSettingUtils.java:1198` |
| K8-only envs/behaviors | Yes | No | No | K8 branch: `PluginSettingUtils.java:1275`; snapshot mode: `PluginSettingUtils.java:1286`; artifact file: `PluginSettingUtils.java:1311` |
| VM-only envs/behaviors | No | Yes | Yes (via VM path) | VM branch: `PluginSettingUtils.java:1312`; daemon off: `PluginSettingUtils.java:1315` |

## UI Field / Input

| Field / input | KubernetesDirect | KubernetesHosted | Cloud | VM | Docker | Optional? | Feature-flag impact | License impact | Tooltip ID | Tooltip content |
|---|---:|---:|---:|---:|---:|---|---|---|---|---|
| `identifier` | Y | Y | Y | Y | Y | No (Required) | None | None | None | None |
| `name` | Y | Y | Y | Y | Y | No (Required) | None | None | None | None |
| `spec.connectorRef` | Y | Y | Y | Y | Y | No (Required) | None | None | None *(Edit)* / `ecrConnector` *(Input Set)* | `ecrConnector` text not present in loaded tooltip dictionary |
| `spec.region` | Y | Y | Y | Y | Y | No (Required) | None | None | `region` | AWS region guidance for ECR/S3/cache steps |
| `spec.account` | Y | Y | Y | Y | Y | No (Required) | None | None | `ecrAccount` | AWS account ID to push image |
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

- The **connector auth mappings** (manual creds, OIDC, cross-account) are applied on both K8 and VM/hosted flows.
- The **step’s plugin env inputs** are mostly shared, but there are infra-specific envs in the K8 and VM branches.
