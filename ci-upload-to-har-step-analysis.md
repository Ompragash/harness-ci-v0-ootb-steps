# Upload Artifacts to Harness Artifact Registry: Inputs & Connector Mapping

This document traces the CI Manager implementation of the **Upload artifacts to Harness Artifact Registry** step, including step inputs, connector/auth behavior, connector → plugin mapping, and infra-type applicability.

All code locations below are workspace-relative `path:line` references.

type: HarUpload
Plugin used: harness/har-plugin:1.0.0 (K8, VM, Hosted/DLITE_VM)

## Known Gaps / Gotchas

- The step model intentionally does not expose `connectorRef`; it uses `registryRef` via `getHARRegistryRef()`.
- `879-pipeline-ci-commons/src/main/java/io/harness/beans/steps/stepinfo/UploadToHarStepInfo.java:124`
- `879-pipeline-ci-commons/src/main/java/io/harness/beans/steps/stepinfo/UploadToHarStepInfo.java:132`
- Step palette visibility and HAR connector resolution are gated by different FFs:
- `HAR_UPLOAD_STEP` controls step visibility.
- `HAR_ENABLED` controls HAR registry connector resolution paths.
- `332-ci-manager/service/src/main/java/io/harness/ci/execution/plan/creator/CIPipelineServiceInfoProvider.java:364`
- `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/integrationstage/k8s/K8InitializeStepUtils.java:2219`
- `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/serializer/vm/VmPluginCompatibleStepSerializer.java:383`

## Phase 1: Step Definition & Executor

Step registration and display metadata:
- `332-ci-manager/service/src/main/java/io/harness/ci/execution/plan/creator/CIPipelineServiceInfoProvider.java:359`

Step node wiring to the spec:
- `332-ci-manager/service/src/main/java/io/harness/ci/beans/steps/nodes/HarUploadNode.java:35`
- `332-ci-manager/service/src/main/java/io/harness/ci/beans/steps/nodes/HarUploadNode.java:44`

Plan creator:
- `332-ci-manager/service/src/main/java/io/harness/ci/execution/plancreator/HarUploadStepPlanCreator.java:19`

Executor (engine step):
- `332-ci-manager/service/src/main/java/io/harness/ci/execution/registrars/ExecutionRegistrar.java:105`
- `332-ci-manager/service/src/main/java/io/harness/ci/execution/states/UploadToHarStep.java:16`

User-facing spec model (inputs live here):
- `879-pipeline-ci-commons/src/main/java/io/harness/beans/steps/stepinfo/UploadToHarStepInfo.java:53`

Default retry value:
- `879-pipeline-ci-commons/src/main/java/io/harness/beans/steps/stepinfo/UploadToHarStepInfo.java:54`

Step env assembly entry point for HAR upload:
- `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:346`
- `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:2038`

## Phase 2: Connector/Auth Analysis

### Connector model used by this step

This step uses an internal Harness Artifact Registry connector path, implemented as a generated `DockerConnectorDTO` with JWT-backed user/password auth:
- `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/buildstate/ConnectorUtils.java:329`
- `954-connector-beans/src/main/java/io/harness/connector/utils/HarnessRegistryConnectorUtils.java:36`

Generated auth type on the internal connector:
- `DockerAuthType.USER_PASSWORD`
- `954-connector-beans/src/main/java/io/harness/connector/utils/HarnessRegistryConnectorUtils.java:47`

Docker auth schema:
- `954-connector-beans/src/main/java/io/harness/delegate/beans/connector/docker/DockerAuthType.java:18`
- `954-connector-beans/src/main/java/io/harness/delegate/beans/connector/docker/DockerUserNamePasswordDTO.java:27`

### Requested AWS auth methods applicability

| Requested Auth Method | Applicable for HarUpload? | Notes |
|---|---:|---|
| Access Key + Secret Key | No | HarUpload does not use AWS connector auth models |
| OIDC / Web Identity | No | Not used by this step’s connector flow |
| IAM Role (assume role) | No | Not used by this step’s connector flow |
| IRSA | No | Not used by this step’s connector flow |
| Internal HAR JWT token | Yes | Primary auth path used by HarUpload |

### Required fields for the active auth flow

Step-level required fields:
- `registryRef`, `packageType`, `sourcePath`
- `879-pipeline-ci-commons/src/main/java/io/harness/beans/steps/stepinfo/UploadToHarStepInfo.java:72`
- `879-pipeline-ci-commons/src/main/java/io/harness/beans/steps/stepinfo/UploadToHarStepInfo.java:78`
- `879-pipeline-ci-commons/src/main/java/io/harness/beans/steps/stepinfo/UploadToHarStepInfo.java:79`

Service config required for token generation:
- `harnessRegistryConfig.jwtSecret`
- `harnessRegistryConfig.httpClientConfig.baseUrl`
- `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:2098`
- `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:2102`

Internally generated connector credential fields:
- docker username = `HARNESS_INTERNAL`
- docker passwordRef = generated JWT token
- `954-connector-beans/src/main/java/io/harness/connector/utils/HarnessRegistryConnectorUtils.java:33`
- `954-connector-beans/src/main/java/io/harness/connector/utils/HarnessRegistryConnectorUtils.java:49`

## Phase 3: Connector → Plugin Input Mapping

### Mapping chain overview

HAR registry reference extraction:
- `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:669`

K8 connector conversion path (includes `registryRef` when `HAR_ENABLED`):
- `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/integrationstage/k8s/K8InitializeStepUtils.java:2202`
- `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/integrationstage/k8s/K8InitializeStepUtils.java:2219`
- `332-ci-manager/service/src/main/java/io/harness/ci/execution/integrationstage/K8InitializeTaskParamsBuilder.java:617`

VM connector resolution path (HAR registry when connectorRef is empty):
- `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/serializer/vm/VmPluginCompatibleStepSerializer.java:256`
- `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/serializer/vm/VmPluginCompatibleStepSerializer.java:278`
- `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/serializer/vm/VmPluginCompatibleStepSerializer.java:382`

### Auth flow details

```text
Auth Method: Internal HAR JWT (Primary)
├── Connector Fields Used: registryRef (step), harnessRegistryConfig.httpClientConfig.baseUrl, harnessRegistryConfig.jwtSecret
├── Mapped to Plugin Env/Args: PLUGIN_REGISTRY, PLUGIN_PKG_URL, PLUGIN_TOKEN, PLUGIN_ACCOUNT, PLUGIN_ORG, PLUGIN_PROJECT
└── Code Location: PluginSettingUtils.java:2046, 2078, 2109, 2077, 2083, 2084
```

```text
Auth Method: Internal Docker USER_PASSWORD (Generated Connector Object)
├── Connector Fields Used: dockerRegistryUrl, auth.credentials.username (HARNESS_INTERNAL), auth.credentials.passwordRef (JWT)
├── Mapped to Plugin Env/Args: secret-env map defines PLUGIN_URL, PLUGIN_USERNAME, PLUGIN_PASSW
└── Code Location: HarnessRegistryConnectorUtils.java:49, PluginSettingUtils.java:745
```

Note on the secondary mapping path:
- HarUpload secret-env map uses `ARTIFACTORY_*` enum keys:
- `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:745`
- Docker connector secret extraction reads `DOCKER_*` enum keys:
- `933-ci-commons/src/main/java/io/harness/connector/ConnectorEnvVariablesHelper.java:451`
- `933-ci-commons/src/main/java/io/harness/connector/ConnectorEnvVariablesHelper.java:481`
- In practice, HarUpload authentication is guaranteed through `PLUGIN_TOKEN` from `getUploadToHarStepInfoEnvVariables(...)`.

## Phase 4: Plugin Inputs Summary (All Inputs)

Primary HAR upload env assembly:
- `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:2038`

Configured plugin images:
- K8 step config image: `332-ci-manager/config/ci-manager-config.yml:466`
- VM image config: `332-ci-manager/config/ci-manager-config.yml:625`
- Runtime image selection for `UPLOAD_HAR`: `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/execution/CIExecutionConfigServiceImpl.java:1072`

| Input Name | Type | Required | Description | Maps To (Plugin) |
|---|---|---:|---|---|
| registryRef | string | Yes | HAR registry reference | `PLUGIN_REGISTRY` (`PluginSettingUtils.java:2046`) |
| packageType | string | Yes | Package type to publish | `PLUGIN_PACKAGE_TYPE` (`PluginSettingUtils.java:2052`) |
| sourcePath | string | Yes | Local source path to upload | `PLUGIN_SOURCE` (`PluginSettingUtils.java:2069`) |
| packageName | string | No | Optional package name override | `PLUGIN_NAME` (`PluginSettingUtils.java:2055`) |
| packageVersion | string | No | Optional package version override | `PLUGIN_VERSION` (`PluginSettingUtils.java:2060`) |
| pomFilePath | string | No | Optional pom file path | `PLUGIN_POM_FILE` (`PluginSettingUtils.java:2065`) |
| settings | map<string,string> | No | Additional plugin env entries | merged directly via `map.putAll(resolvedSettings)` (`PluginSettingUtils.java:2088`) |
| runAsUser | integer | No | Container user override | Runtime container setting (`VmPluginCompatibleStepSerializer.java:237`) |
| resources | object | No | Container resources | Runtime container setting (`UploadToHarStepInfo.java:75`) |
| retry | integer | No | Step retry count | Defaults to `1` (`UploadToHarStepInfo.java:54`) |
| (internal) command | string | Always | HAR upload command mode | `PLUGIN_COMMAND=push` (`PluginSettingUtils.java:2043`) |
| (internal) description | string | Always | Default description | `PLUGIN_DESCRIPTION` (`PluginSettingUtils.java:2073`) |
| (internal) account/org/project | string | Always | Scope context for upload | `PLUGIN_ACCOUNT`, `PLUGIN_ORG`, `PLUGIN_PROJECT` (`PluginSettingUtils.java:2077`) |
| (internal) HAR service URL | string | Always | HAR package service base URL | `PLUGIN_PKG_URL` (`PluginSettingUtils.java:2078`) |
| (internal) HAR auth token | string | Always | JWT auth token for HAR upload | `PLUGIN_TOKEN` (`PluginSettingUtils.java:2117`) |

### Auth Flow Matrix

| Auth Method | Connector Fields | Plugin Env Vars | Notes |
|---|---|---|---|
| Internal HAR JWT (primary) | `registryRef`; service config `jwtSecret` + HAR base URL | `PLUGIN_REGISTRY`, `PLUGIN_PKG_URL`, `PLUGIN_TOKEN`, `PLUGIN_ACCOUNT`, `PLUGIN_ORG`, `PLUGIN_PROJECT` | Primary auth path in HarUpload env builder |
| Generated Docker USER_PASSWORD connector | `dockerRegistryUrl`, generated `username/passwordRef` | Mapping table defines `PLUGIN_URL`, `PLUGIN_USERNAME`, `PLUGIN_PASSW` | Mapping exists in step secret-env map; token-based auth remains primary |

## Infra Applicability (K8 vs VM vs Hosted/DLITE)

Infra types:
- `K8`, `VM`, `DLITE_VM`
- `879-pipeline-ci-commons/src/main/java/io/harness/beans/sweepingoutputs/StageInfraDetails.java:33`

K8 path calls plugin env builder:
- `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/serializer/PluginCompatibleStepSerializer.java:76`

VM/Hosted path calls VM runtime env builder:
- `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/serializer/vm/VmPluginCompatibleStepSerializer.java:92`

### Applicability Matrix

| Concern | K8 | VM | Hosted/DLITE_VM | Notes |
|---|---:|---:|---:|---|
| HarUpload step type + same plugin family | Yes | Yes | Yes | Image config maps to `harness/har-plugin:1.0.0` on both K8 + VM configs |
| `registryRef` → HAR connector resolution | Yes | Yes | Yes (via VM path) | K8 uses `registryRef` conversion; VM uses `isHarnessRegistryStep(...)` fallback |
| HAR token env generation (`PLUGIN_TOKEN`) | Yes | Yes | Yes | Generated in shared HAR env builder (`PluginSettingUtils.java:2109`) |
| User-provided `connectorRef` usage | No | No | No | Step model returns empty connectorRef (`UploadToHarStepInfo.java:128`) |

## UI Field / Input:

| Field / Input | Visible In Build Flow(s) | Optional? | Tooltip ID | Tooltip Text / Content | Placeholder | Feature-Flag Influence | License Influence |
|---|---|---|---|---|---|---|---|
| `identifier` | All | No | — | — | — | None | None |
| `name` | All | No | `harUpload_name` (generated by form context) | Not found in current `ng-tooltip` dictionary export | — | None | None |
| `description` | All | Yes | `harUpload_description` (generated by form context) | Not found in current `ng-tooltip` dictionary export | — | None | None |
| `spec.packageType` | All | No | — | — | `pipeline.packageTypePlaceholder` | None | None |
| `spec.registryRef` | All | No | `harUpload_spec.registryRef` (generated by form context) | Not found in current `ng-tooltip` dictionary export | `- common.selectRegistry -` | None | None |
| `spec.packageName` (rendered from `spec.repo` with `fieldName='packageName'`) | All, only when `packageType = GENERIC` | Conditional | — | — | — | None | None |
| `spec.packageVersion` | All, when `packageType = GENERIC` or `GO` | Conditional | — | — | — | None | None |
| `spec.sourcePath` | All | No | `sourcePath` | Path to the artifact file/folder to upload. | — | None | None |
| `spec.settings` (optional config map) | All | Yes | — (edit view) | — | — | None | None |
| `spec.runAsUser` (optional config) | `Cloud`, `KubernetesDirect` | Yes | `runAsUser` | Specify the user ID to run processes in pod/containers. | `1000` | None | None |
| `spec.limitMemory` (optional config UI field) | `KubernetesDirect`, `KubernetesHosted` | Yes | `limitMemory` | Max container memory (default `500Mi`). | — | None | None |
| `spec.limitCPU` (optional config UI field) | `KubernetesDirect`, `KubernetesHosted` | Yes | `limitCPULabel` | Max container CPU (default `400m`). | — | None | None |
| `timeout` | All | Yes | — (edit view) | — | `common.durationPlaceholder` | None | None |

Notes:
- `packageName` required only for `GENERIC`.
- `packageVersion` required for `GENERIC` and `GO`.
- For `GO`, `packageVersion` also shows helper text (`har.formFields.tagSelectField.goVersionHelperText`).
- In runtime/input-set mode, runtime-enabled fields can show `<+input>`.

## Key Takeaways

- `HarUpload` uses an internal HAR registry flow, not AWS connector auth flows.
- Primary auth for upload is a server-generated HAR JWT token mapped to `PLUGIN_TOKEN`.
- Inputs are stable across K8, VM, and hosted (DLITE_VM); plugin image differs only by infra config source, not step schema.
