# Background Step: Inputs & Connector Mapping

This document traces the CI Manager implementation of the **Background** step, its user-facing inputs, connector/auth handling, connector → env mapping, and infra-type applicability.

All code locations below are workspace-relative `path:line` references.

type: Background
Plugin used: user-provided `image` (background service container)

## Known Gaps / Gotchas

- **K8 requires `image` and `connectorRef`** (unless `harnessManagedImage`), and **disallows `portBindings`**. Evidence: `K8InitializeStepUtils.java:1311`, `K8InitializeStepUtils.java:1318`, `K8InitializeStepUtils.java:1326`.
- If `command` is set and `entrypoint` is empty on VM, the serializer **injects a shell entrypoint** and **wraps the command** with an early-exit prefix. Evidence: `VmBackgroundStepSerializer.java:95`, `VmBackgroundStepSerializer.java:121`.

## Phase 1: Step Definition & Executor

Step registration and display metadata:
- `332-ci-manager/service/src/main/java/io/harness/ci/execution/plan/creator/CIPipelineServiceInfoProvider.java:267`

Step node wiring to the spec:
- `877-pipeline-ci-cd-commons/src/java/io/harness/beans/steps/nodes/BackgroundStepNode.java:41`
- `877-pipeline-ci-cd-commons/src/java/io/harness/beans/steps/nodes/BackgroundStepNode.java:53`

Plan creators:
- V0: `332-ci-manager/service/src/main/java/io/harness/ci/execution/plancreator/BackgroundStepPlanCreator.java:17`
- V1: `332-ci-manager/service/src/main/java/io/harness/ci/execution/plancreator/V1/BackgroundStepPlanCreatorV1.java:17`

Executor (engine step):
- `332-ci-manager/service/src/main/java/io/harness/ci/execution/registrars/ExecutionRegistrar.java:87`
- `332-ci-manager/service/src/main/java/io/harness/ci/execution/states/BackgroundStep.java:16`

User-facing spec model (inputs live here):
- `879-pipeline-ci-commons/src/main/java/io/harness/beans/steps/stepinfo/BackgroundStepInfo.java:61`

Default retry value:
- `879-pipeline-ci-commons/src/main/java/io/harness/beans/steps/stepinfo/BackgroundStepInfo.java:62`

K8 container definition for Background steps:
- `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/integrationstage/k8s/K8InitializeStepUtils.java:1307`

VM/Hosted serializer for Background steps:
- `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/serializer/vm/VmBackgroundStepSerializer.java:66`

Lite-engine step serialization:
- `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/serializer/BackgroundStepProtobufSerializer.java:60`

## Phase 2: Connector Analysis

### Connector Types Used

The Background step uses an **image connector** for pulling the background container image.
- K8 requires `connectorRef` unless the image is harness-managed: `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/integrationstage/k8s/K8InitializeStepUtils.java:1318`
- VM/Hosted resolves connector from `registries` or `connectorRef`: `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/serializer/vm/VmBackgroundStepSerializer.java:76`

### Docker Connector Auth Methods (used for image pulls)

Supported Docker auth types:
- `USER_PASSWORD`, `ANONYMOUS`: `954-connector-beans/src/main/java/io/harness/delegate/beans/connector/docker/DockerAuthType.java:18`

Username + Password requirements: `dockerRegistryUrl`, `providerType`, `auth`: `954-connector-beans/src/main/java/io/harness/delegate/beans/connector/DockerConnectorDTO.java:56`; `username` or `usernameRef` (one-of), `passwordRef`: `954-connector-beans/src/main/java/io/harness/delegate/beans/connector/docker/DockerUserNamePasswordDTO.java:27`.

Anonymous requirements: `auth.type=ANONYMOUS` required: `954-connector-beans/src/main/java/io/harness/delegate/beans/connector/docker/DockerAuthenticationDTO.java:34`; anonymous auth returns no decryptable entities: `954-connector-beans/src/main/java/io/harness/delegate/beans/connector/DockerConnectorDTO.java:79`.

## Phase 3: Connector → Env Mapping

The Background step **does not map connector secrets to `PLUGIN_*` env vars**. Connectors are only used for image pulls.

```
Auth Method: Docker Username/Password
├── Connector Fields Used: dockerRegistryUrl, auth(type=USER_PASSWORD), username/usernameRef, passwordRef
├── Mapped to Plugin Env/Args: None (connector used for image pull)
└── Code Location: VmBackgroundStepSerializer.java:126 (image connector wiring)
```

```
Auth Method: Docker Anonymous
├── Connector Fields Used: dockerRegistryUrl, auth(type=ANONYMOUS)
├── Mapped to Plugin Env/Args: None (connector used for image pull)
└── Code Location: VmBackgroundStepSerializer.java:126
```

### Auth Flow Matrix

| Auth Method | Connector Fields | Plugin Env Vars | Notes |
|---|---|---|---|
| Docker Username/Password | `dockerRegistryUrl`, `auth.type=USER_PASSWORD`, `username|usernameRef`, `passwordRef` | None | Image pull connector only; no env mapping. |
| Docker Anonymous | `dockerRegistryUrl`, `auth.type=ANONYMOUS` | None | No decryptables for anonymous auth. |

## Phase 4: Background Step Inputs Summary

| Input Name | Type | Required | Description | Maps To (Runtime) |
|---|---|---|---|---|
| `image` | string | **Required for K8** | Background container image. | K8: `K8InitializeStepUtils.java:1311`. VM: `VmBackgroundStepSerializer.java:71`. |
| `connectorRef` | string | **Required for K8** unless `harnessManagedImage` | Image pull connector. | K8: `K8InitializeStepUtils.java:1318`. VM: `VmBackgroundStepSerializer.java:76`. |
| `command` | string | Optional | Command to execute in background container. | VM: `VmBackgroundStepSerializer.java:69`. Lite-engine: `BackgroundStepProtobufSerializer.java:78`. |
| `entrypoint` | list<string> | Optional | Override container entrypoint. | VM: `VmBackgroundStepSerializer.java:90`. Lite-engine: `BackgroundStepProtobufSerializer.java:125`. |
| `envVariables` | map<string,string> | Optional | Environment variables. | K8: `K8InitializeStepUtils.java:1344`. VM: `VmBackgroundStepSerializer.java:101`. Lite-engine: `BackgroundStepProtobufSerializer.java:86`. |
| `resources` | object | Optional | CPU/memory requests & limits. | K8: `K8InitializeStepUtils.java:1369`. |
| `privileged` | boolean | Optional | Run container in privileged mode. | K8: `K8InitializeStepUtils.java:1374`. VM: `VmBackgroundStepSerializer.java:93`. |
| `runAsUser` | int | Optional | Run container as specific UID. | K8: `K8InitializeStepUtils.java:1350`. VM: `VmBackgroundStepSerializer.java:156`. |
| `shell` | enum | Optional | Shell type (affects entrypoint/command). | VM entrypoint calc: `VmBackgroundStepSerializer.java:95`. Lite-engine shell: `BackgroundStepProtobufSerializer.java:111`. |
| `imagePullPolicy` | enum | Optional | Image pull policy. | K8: `K8InitializeStepUtils.java:1376`. VM: `VmBackgroundStepSerializer.java:92`. |
| `portBindings` | map<string,string> | Optional (VM only) | Host-to-container port mapping. | VM: `VmBackgroundStepSerializer.java:83`. **K8 forbidden**: `K8InitializeStepUtils.java:1326`. |
| `ports` | list<string> | Optional (hidden) | Alternative to `portBindings`. | VM: `VmBackgroundStepSerializer.java:84`. |
| `networkAliases` | list<string> | Optional (VM only) | Docker network aliases. | VM: `VmBackgroundStepSerializer.java:143`. |
| `reports` | UnitTestReport | Optional | JUnit report capture. | VM: `VmBackgroundStepSerializer.java:147`. Lite-engine: `BackgroundStepProtobufSerializer.java:97`. |
| `retry` | int | Optional (default 1) | Step retry count. | Default: `BackgroundStepInfo.java:62`. |
| `harnessManagedImage` | bool | Optional | Indicates Harness-managed image. | K8 connector requirement uses this: `K8InitializeStepUtils.java:1318`. |

## Infra Applicability (K8 vs VM vs Hosted/DLITE)

Infra types for CI steps are:
- `K8`, `VM`, and `DLITE_VM`: `879-pipeline-ci-commons/src/main/java/io/harness/beans/sweepingoutputs/StageInfraDetails.java:33`

K8 path uses the Background container definition:
- `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/integrationstage/k8s/K8InitializeStepUtils.java:1307`

VM/Hosted path uses the Background VM serializer:
- `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/serializer/vm/VmBackgroundStepSerializer.java:66`

### Applicability Matrix

| Concern | K8 | VM | Hosted/DLITE_VM | Notes |
|---|---:|---:|---:|---|
| `image` required | Yes | Yes (practically) | Yes | K8 enforces image required. |
| `connectorRef` required | Yes (unless harness-managed) | Optional | Optional | K8 enforces unless `harnessManagedImage`. |
| `portBindings` | No | Yes | Yes | K8 rejects `portBindings`. |
| `networkAliases` | No | Yes | Yes | VM-only. |
| `command`/`entrypoint` | Yes | Yes | Yes | Wired via VM serializer and lite-engine protobuf. |

## Key Takeaways

- Background is **image-driven** and uses the connector only for pulling that image (no env mapping).
- K8 has stricter requirements: **image + connectorRef required**, and **portBindings are invalid**.
- VM/Hosted supports port bindings, network aliases, and command/entrypoint customization.
