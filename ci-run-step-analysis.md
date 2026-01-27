# Run Step: Inputs & Connector Mapping

This document traces the CI Manager implementation of the **Run** step, its user-facing inputs, connector/auth handling, connector → env mapping, and infra-type applicability.

All code locations below are workspace-relative `path:line` references.

type: Run
Plugin used: user-provided `image` (or command-only containerless execution on VM/Hosted)

## Known Gaps / Gotchas

- **K8 requires `image`** and **requires `connectorRef` or `registryRef`**.
  - `image` required: `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/integrationstage/k8s/K8InitializeStepUtils.java:1025`
  - `connectorRef`/`registryRef` required: `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/integrationstage/k8s/K8InitializeStepUtils.java:1032`
- VM/Hosted throws if **both `command` and `image` are empty**.
  - Validation: `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/serializer/vm/VmRunStepSerializer.java:85`
- When **no command** is provided in K8 lite-engine serialization, the serializer expects `connectorRef` or `registryRef` if `image` is present.
  - Validation: `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/serializer/RunStepProtobufSerializer.java:194`

## Phase 1: Step Definition & Executor

Step registration and display metadata:
- `332-ci-manager/service/src/main/java/io/harness/ci/execution/plan/creator/CIPipelineServiceInfoProvider.java:252`

Step node wiring to the spec:
- `877-pipeline-ci-cd-commons/src/java/io/harness/beans/steps/nodes/RunStepNode.java:40`
- `877-pipeline-ci-cd-commons/src/java/io/harness/beans/steps/nodes/RunStepNode.java:50`

Plan creators:
- V0: `332-ci-manager/service/src/main/java/io/harness/ci/execution/plancreator/RunStepPlanCreator.java:20`
- V1: `332-ci-manager/service/src/main/java/io/harness/ci/execution/plancreator/V1/RunStepPlanCreatorV1.java:20`

Executor (engine step):
- `332-ci-manager/service/src/main/java/io/harness/ci/execution/registrars/ExecutionRegistrar.java:84`
- `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/states/RunStep.java:27`

User-facing spec model (inputs live here):
- `879-pipeline-ci-commons/src/main/java/io/harness/beans/steps/stepinfo/RunStepInfo.java:65`

Default retry value:
- `879-pipeline-ci-commons/src/main/java/io/harness/beans/steps/stepinfo/RunStepInfo.java:66`

K8 container definition for Run steps:
- `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/integrationstage/k8s/K8InitializeStepUtils.java:1021`

VM/Hosted serializer for Run steps:
- `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/serializer/vm/VmRunStepSerializer.java:72`

Lite-engine step serialization:
- `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/serializer/RunStepProtobufSerializer.java:80`

## Phase 2: Connector Analysis

### Connector Types Used

The Run step uses an **image connector** for pulling the run container image.
- K8 requires `connectorRef` or `registryRef`: `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/integrationstage/k8s/K8InitializeStepUtils.java:1032`
- VM/Hosted resolves connector from `registries` or `connectorRef`, with optional `registryRef` (HAR): `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/serializer/vm/VmRunStepSerializer.java:89`

### Docker Connector Auth Methods (used for image pulls)

Supported Docker auth types:
- `USER_PASSWORD`, `ANONYMOUS`: `954-connector-beans/src/main/java/io/harness/delegate/beans/connector/docker/DockerAuthType.java:18`

Username + Password requirements: `dockerRegistryUrl`, `providerType`, `auth`: `954-connector-beans/src/main/java/io/harness/delegate/beans/connector/DockerConnectorDTO.java:56`; `username` or `usernameRef` (one-of), `passwordRef`: `954-connector-beans/src/main/java/io/harness/delegate/beans/connector/docker/DockerUserNamePasswordDTO.java:27`.

Anonymous requirements: `auth.type=ANONYMOUS` required: `954-connector-beans/src/main/java/io/harness/delegate/beans/connector/docker/DockerAuthenticationDTO.java:34`; anonymous auth returns no decryptable entities: `954-connector-beans/src/main/java/io/harness/delegate/beans/connector/DockerConnectorDTO.java:79`.

## Phase 3: Connector → Env Mapping

The Run step **does not map connector secrets to `PLUGIN_*` env vars**. Connectors are only used for image pulls.

```
Auth Method: Docker Username/Password
├── Connector Fields Used: dockerRegistryUrl, auth(type=USER_PASSWORD), username/usernameRef, passwordRef
├── Mapped to Plugin Env/Args: None (connector used for image pull)
└── Code Location: VmRunStepSerializer.java:211 (image connector wiring)
```

```
Auth Method: Docker Anonymous
├── Connector Fields Used: dockerRegistryUrl, auth(type=ANONYMOUS)
├── Mapped to Plugin Env/Args: None (connector used for image pull)
└── Code Location: VmRunStepSerializer.java:211
```

### Auth Flow Matrix

| Auth Method | Connector Fields | Plugin Env Vars | Notes |
|---|---|---|---|
| Docker Username/Password | `dockerRegistryUrl`, `auth.type=USER_PASSWORD`, `username|usernameRef`, `passwordRef` | None | Image pull connector only; no env mapping. |
| Docker Anonymous | `dockerRegistryUrl`, `auth.type=ANONYMOUS` | None | No decryptables for anonymous auth. |

## Phase 4: Run Step Inputs Summary

| Input Name | Type | Required | Description | Maps To (Runtime) |
|---|---|---|---|---|
| `command` | string | Optional | Command to execute. | VM: `VmRunStepSerializer.java:75`. K8 lite-engine: `RunStepProtobufSerializer.java:99`. |
| `image` | string | **Required for K8** | Container image to run. | K8: `K8InitializeStepUtils.java:1025`. VM: `VmRunStepSerializer.java:77`. |
| `connectorRef` | string | **Required for K8** unless `registryRef` provided | Image pull connector. | K8: `K8InitializeStepUtils.java:1032`. VM: `VmRunStepSerializer.java:92`. |
| `registryRef` | string | Optional | HAR registry reference for image pull. | K8: `K8InitializeStepUtils.java:1078`. VM: `VmRunStepSerializer.java:170`. Lite-engine usage when no command: `RunStepProtobufSerializer.java:196`. |
| `envVariables` | map<string,string> | Optional | Environment variables. | K8: `K8InitializeStepUtils.java:1055`. VM: `VmRunStepSerializer.java:98`. |
| `resources` | object | Optional | CPU/memory requests & limits. | K8: `K8InitializeStepUtils.java:1083`. |
| `privileged` | boolean | Optional | Run container in privileged mode. | K8: `K8InitializeStepUtils.java:1088`. VM: `VmRunStepSerializer.java:219`. |
| `runAsUser` | int | Optional | Run container as specific UID. | K8: `K8InitializeStepUtils.java:1060`. VM: `VmRunStepSerializer.java:212`. |
| `shell` | enum | Optional | Shell type (affects entrypoint/command). | VM: `VmRunStepSerializer.java:104`. K8 lite-engine shell: `RunStepProtobufSerializer.java:174`. |
| `imagePullPolicy` | enum | Optional | Image pull policy. | K8: `K8InitializeStepUtils.java:1090`. VM: `VmRunStepSerializer.java:168`. |
| `outputVariables` | list | Optional | Exported output variables. | K8: `RunStepProtobufSerializer.java:151`. VM: `VmRunStepSerializer.java:135`. |
| `reports` | UnitTestReport | Optional | JUnit report capture. | K8: `RunStepProtobufSerializer.java:139`. VM: `VmRunStepSerializer.java:232`. |
| `outputAlias` | object | Optional | Alias for exported output variables. | Applied in `AbstractStepExecutable`: `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/states/AbstractStepExecutable.java:702`. |
| `retry` | int | Optional (default 1) | Step retry count. | Default: `RunStepInfo.java:66`. |

## Infra Applicability (K8 vs VM vs Hosted/DLITE)

Infra types for CI steps are:
- `K8`, `VM`, and `DLITE_VM`: `879-pipeline-ci-commons/src/main/java/io/harness/beans/sweepingoutputs/StageInfraDetails.java:33`

K8 path uses the Run container definition and lite-engine serialization:
- `K8InitializeStepUtils.java:1021`
- `RunStepProtobufSerializer.java:80`

VM/Hosted path uses the Run VM serializer:
- `VmRunStepSerializer.java:72`

### Applicability Matrix

| Concern | K8 | VM | Hosted/DLITE_VM | Notes |
|---|---:|---:|---:|---|
| `image` required | Yes | No (command-only allowed) | No (command-only allowed) | K8 enforces image required. |
| `connectorRef` or `registryRef` required | Yes | Only if image provided | Only if image provided | K8 enforces connector/registry for image. |
| Command-only run | No | Yes | Yes | VM allows command-only; K8 requires image. |
| Output variables | Yes | Yes | Yes | Added as env-var outputs and outputs. |

## Key Takeaways

- Run is **image-driven on K8**, and **command-only is allowed on VM/Hosted**.
- Connectors are used **only for pulling the image**; there is no connector → `PLUGIN_*` env mapping.
- Output variables and optional output aliases are supported across infra types.
