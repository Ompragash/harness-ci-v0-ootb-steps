# Plugin Step: Inputs & Connector Mapping

This document traces the CI Manager implementation of the **Plugin** step, its user-facing inputs, connector/auth handling, connector → plugin env mapping, and infra-type applicability.

All code locations below are workspace-relative `path:line` references.

type: Plugin
Plugin used: user-provided `image` (or `uses` for hosted/DLITE cloud builds)

## Known Gaps / Gotchas

- `connectorRef` is **optional** for Plugin steps, and there is **no default connector → PLUGIN_* secret mapping** for the PLUGIN step type.
  - `isConnectorMandatory()` returns false: `879-pipeline-ci-commons/src/main/java/io/harness/beans/steps/stepinfo/PluginStepInfo.java:151`
  - `PLUGIN` returns an empty mapping: `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:743`
- `settings` → `PLUGIN_*` env var mapping is wired for the VM/Hosted serializer, but **not used in the K8 container definition** for Plugin steps.
  - VM env mapping (settings → `PLUGIN_*`): `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/serializer/vm/VmPluginStepSerializer.java:110`
  - K8 Plugin container definition only uses `envVariables`, not `settings`: `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/integrationstage/k8s/K8InitializeStepUtils.java:1447`
- `uses` is **only supported for hosted/DLITE_VM**. For VM/K8, `uses` throws an error, and if both `image` and `uses` are set, `uses` is ignored.
  - `uses` is DLITE_VM-only and produces a containerless entrypoint: `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/serializer/vm/VmPluginStepSerializer.java:264`
  - If both `image` and `uses` are set, `uses` is ignored: `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/serializer/vm/VmPluginStepSerializer.java:178`
- `reports` (JUnit) are wired in the VM serializer but **not used** in the K8 Plugin container definition.
  - VM report wiring: `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/serializer/vm/VmPluginStepSerializer.java:329`
  - K8 container definition has no report wiring: `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/integrationstage/k8s/K8InitializeStepUtils.java:1441`

## Phase 1: Step Definition & Executor

Step registration and display metadata:
- `332-ci-manager/service/src/main/java/io/harness/ci/execution/plan/creator/CIPipelineServiceInfoProvider.java:274`

Step node wiring to the spec:
- `879-pipeline-ci-commons/src/main/java/io/harness/beans/steps/nodes/PluginStepNode.java:41`
- `879-pipeline-ci-commons/src/main/java/io/harness/beans/steps/nodes/PluginStepNode.java:53`

Plan creators:
- V0: `332-ci-manager/service/src/main/java/io/harness/ci/execution/plancreator/PluginStepPlanCreator.java:20`
- V1: `332-ci-manager/service/src/main/java/io/harness/ci/execution/plancreator/V1/PluginStepPlanCreatorV1.java:30`

Executor (engine step):
- `332-ci-manager/service/src/main/java/io/harness/ci/execution/registrars/ExecutionRegistrar.java:88`
- `332-ci-manager/service/src/main/java/io/harness/ci/execution/states/PluginStep.java:25`

User-facing spec model (inputs live here):
- `879-pipeline-ci-commons/src/main/java/io/harness/beans/steps/stepinfo/PluginStepInfo.java:60`

Default retry value:
- `879-pipeline-ci-commons/src/main/java/io/harness/beans/steps/stepinfo/PluginStepInfo.java:66`

K8 container definition for Plugin steps:
- `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/integrationstage/k8s/K8InitializeStepUtils.java:1441`

VM/Hosted serializer for Plugin steps:
- `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/serializer/vm/VmPluginStepSerializer.java:106`

## Phase 2: Connector Analysis

### Connector Types Used

The Plugin step uses an **image connector** for pulling the plugin image (via `connectorRef` or `registryRef`).
- K8 path uses both `connectorRef` and `registryRef` on the container image details: `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/integrationstage/k8s/K8InitializeStepUtils.java:1483`
- VM/Hosted resolves connector from `registries` or `connectorRef`: `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/serializer/vm/VmPluginStepSerializer.java:167`

### Docker Connector Auth Methods (used for image pulls)

Supported Docker auth types:
- `USER_PASSWORD`, `ANONYMOUS`: `954-connector-beans/src/main/java/io/harness/delegate/beans/connector/docker/DockerAuthType.java:18`

Required fields by auth method:

Username + Password:
- `dockerRegistryUrl`, `providerType`, `auth`: `954-connector-beans/src/main/java/io/harness/delegate/beans/connector/DockerConnectorDTO.java:56`
- `username` or `usernameRef` (one-of), `passwordRef`: `954-connector-beans/src/main/java/io/harness/delegate/beans/connector/docker/DockerUserNamePasswordDTO.java:27`

Anonymous:
- `auth.type=ANONYMOUS` required: `954-connector-beans/src/main/java/io/harness/delegate/beans/connector/docker/DockerAuthenticationDTO.java:34`
- Anonymous auth returns no decryptable entities: `954-connector-beans/src/main/java/io/harness/delegate/beans/connector/DockerConnectorDTO.java:79`

### HAR (Harness Artifact Registry) Image Pulls

When `registryRef` is set and the image is detected as a Harness registry image, VM/Hosted will use the HAR connector and may inject OIDC envs:
- HAR detection & image FQN: `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/serializer/vm/VmPluginStepSerializer.java:307`
- OIDC env injection for registry pulls: `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/serializer/vm/VmPluginStepSerializer.java:311`

## Phase 3: Connector → Plugin Input Mapping

For the Plugin step, connector secrets are **not mapped** to `PLUGIN_*` env vars by default. Instead, connectors are used to **pull the plugin image**.

```
Auth Method: Docker Username/Password
├── Connector Fields Used: dockerRegistryUrl, auth(type=USER_PASSWORD), username/usernameRef, passwordRef
├── Mapped to Plugin Env/Args: None (connector used for image pull; no PLUGIN_* mapping)
└── Code Location: VmPluginStepSerializer.java:318 (image connector wiring)
```

```
Auth Method: Docker Anonymous
├── Connector Fields Used: dockerRegistryUrl, auth(type=ANONYMOUS)
├── Mapped to Plugin Env/Args: None (connector used for image pull; no PLUGIN_* mapping)
└── Code Location: VmPluginStepSerializer.java:318
```

```
Auth Method: HAR Registry
├── Connector Fields Used: registryRef (HAR)
├── Mapped to Plugin Env/Args: None (image pull via HAR connector; OIDC env injection may occur)
└── Code Location: VmPluginStepSerializer.java:307
```

### Auth Flow Matrix

| Auth Method | Connector Fields | Plugin Env Vars | Notes |
|---|---|---|---|
| Docker Username/Password | `dockerRegistryUrl`, `auth.type=USER_PASSWORD`, `username|usernameRef`, `passwordRef` | None | Image pull connector only; no PLUGIN_* mapping for PLUGIN steps. |
| Docker Anonymous | `dockerRegistryUrl`, `auth.type=ANONYMOUS` | None | No decryptables for anonymous auth. |
| HAR Registry | `registryRef` | None | VM/Hosted uses HAR connector, may inject OIDC envs. |

## Phase 4: Plugin Inputs Summary

| Input Name | Type | Required | Description | Maps To (Plugin) |
|---|---|---|---|---|
| `image` | string | **Required for K8**; required for VM unless `uses` is set | Plugin container image to run. | K8 container image: `K8InitializeStepUtils.java:1468`. VM container image: `VmPluginStepSerializer.java:159`. |
| `uses` | string | Optional (DLITE_VM only) | Hosted cloud builds can reference a Harness plugin repo. | DLITE_VM only; entrypoint is `plugin -kind harness -repo <uses>`: `VmPluginStepSerializer.java:264`. |
| `settings` | map<string, any> | Optional | Plugin `with` settings; mapped to env vars. | VM maps to `PLUGIN_<KEY>` envs: `VmPluginStepSerializer.java:110`. K8 Plugin container definition does **not** map `settings`: `K8InitializeStepUtils.java:1447`. |
| `envVariables` | map<string, string> | Optional | Explicit env vars. | K8 uses `envs`: `K8InitializeStepUtils.java:1455`. VM uses `envVars`: `VmPluginStepSerializer.java:146`. |
| `entrypoint` | list<string> | Optional | Custom entrypoint. | Not wired in K8 container definition; VM only uses entrypoint for containerless `uses` flow: `VmPluginStepSerializer.java:268`. |
| `connectorRef` | string | Optional | Image pull connector for registry credentials. | K8: `K8InitializeStepUtils.java:1483`. VM: `VmPluginStepSerializer.java:167`. |
| `registryRef` | string | Optional | Harness Artifact Registry reference for image pulls. | K8: `K8InitializeStepUtils.java:1486`. VM HAR path: `VmPluginStepSerializer.java:307`. |
| `resources` | object | Optional | CPU/memory requests & limits. | K8 container resources: `K8InitializeStepUtils.java:1490`. |
| `privileged` | boolean | Optional | Run container in privileged mode. | K8: `K8InitializeStepUtils.java:1496`. VM: `VmPluginStepSerializer.java:337`. |
| `runAsUser` | int | Optional | Run container as specific UID. | K8: `K8InitializeStepUtils.java:1460`. VM: `VmPluginStepSerializer.java:338`. |
| `imagePullPolicy` | enum | Optional | Image pull policy. | K8: `K8InitializeStepUtils.java:1498`. VM: `VmPluginStepSerializer.java:275`. |
| `reports` | UnitTestReport | Optional | JUnit report capture. | VM only: `VmPluginStepSerializer.java:329`. |
| `retry` | int | Optional (default 1) | Step retry count. | Default: `PluginStepInfo.java:66`. |

## Infra Applicability (K8 vs VM vs Hosted/DLITE)

Infra types for CI steps are:
- `K8`, `VM`, and `DLITE_VM`: `879-pipeline-ci-commons/src/main/java/io/harness/beans/sweepingoutputs/StageInfraDetails.java:33`

K8 uses the Plugin container definition:
- `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/integrationstage/k8s/K8InitializeStepUtils.java:1441`

VM/Hosted uses the Plugin VM serializer:
- `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/serializer/vm/VmPluginStepSerializer.java:106`

### Applicability Matrix

| Concern | K8 | VM | Hosted/DLITE_VM | Notes |
|---|---:|---:|---:|---|
| `image` required | Yes | Yes (unless `uses`) | Yes (unless `uses`) | K8 `image` is required: `K8InitializeStepUtils.java:1468`. |
| `uses` supported | No | No | Yes | `uses` is DLITE_VM-only: `VmPluginStepSerializer.java:264`. |
| `settings` → `PLUGIN_*` envs | No (not wired) | Yes | Yes | VM mapping: `VmPluginStepSerializer.java:110`. |
| `envVariables` | Yes | Yes | Yes | K8 uses `envs`; VM uses `envVars`. |
| `reports` (JUnit) | No | Yes | Yes | VM-only wiring: `VmPluginStepSerializer.java:329`. |
| Connector secret mapping to `PLUGIN_*` | No | No | No | `PLUGIN` has empty mapping: `PluginSettingUtils.java:743`. |

## Key Takeaways

- The Plugin step is **user-image driven**. There is **no fixed plugin image**; you must supply `image` (or `uses` for DLITE_VM).
- **Connector secrets are not mapped** to `PLUGIN_*` envs for this step; connectors are only used for image pulls.
- There are **infra differences**: `settings` and `reports` are wired in VM/Hosted, but not in the K8 Plugin container definition.
