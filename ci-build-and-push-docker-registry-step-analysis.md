# Build and Push an Image to Docker Registry: Inputs & Docker Connector Mapping

This document traces the CI Manager implementation of the **Build and Push an Image to Docker Registry** step, its user-facing inputs, Docker connector auth flows, connector → plugin env mapping, and infra-type applicability.

All code locations below are workspace-relative `path:line` references.

type: BuildAndPushDockerRegistry
Plugin used: plugins/kaniko:1.13.3 (K8), plugins/buildx:1.3.12 (buildx)

## Known Gaps / Gotchas

- `connectorRef` is `@NotNull` on the spec, but the step treats a blank `connectorRef` plus a `registryRef` as a Harness Artifact Registry (HAR) flow.
  - Required on spec: `879-pipeline-ci-commons/src/main/java/io/harness/beans/steps/stepinfo/DockerStepInfo.java:73`
  - HAR detection uses `registryRef` + blank `connectorRef`: `332-ci-manager/service/src/main/java/io/harness/ci/execution/states/DockerStep.java:132`
- `repo` can be rewritten based on feature flags:
  - HAR registryRef path: `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:1331`
  - FQN rewrite via connector when `CI_REMOVE_FQN_DEPENDENCY_FOR_PRIVATE_REGISTRY_CONNECTOR_DOCKER` is enabled: `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:1335`
- `remoteCacheRepo` is only applied in the K8 branch; VM/Hosted does not use it.
  - K8-only handling: `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:1429`
- Only the first base image connector is used, even if multiple are provided.
  - `// only one baseImageConnector is allowed currently`: `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:776`

## Phase 1: Step Definition & Executor

Step registration and display metadata:
- `332-ci-manager/service/src/main/java/io/harness/ci/execution/plan/creator/CIPipelineServiceInfoProvider.java:367`

Step node wiring to the spec:
- `332-ci-manager/service/src/main/java/io/harness/ci/beans/steps/nodes/BuildAndPushDockerNode.java:35`
- `332-ci-manager/service/src/main/java/io/harness/ci/beans/steps/nodes/BuildAndPushDockerNode.java:45`

Plan creator:
- `332-ci-manager/service/src/main/java/io/harness/ci/execution/plancreator/DockerStepPlanCreator.java:19`

Executor (engine step):
- `332-ci-manager/service/src/main/java/io/harness/ci/execution/registrars/ExecutionRegistrar.java:95`
- `332-ci-manager/service/src/main/java/io/harness/ci/execution/states/DockerStep.java:54`

User-facing spec model (inputs live here):
- `879-pipeline-ci-commons/src/main/java/io/harness/beans/steps/stepinfo/DockerStepInfo.java:52`

Default retry value:
- `879-pipeline-ci-commons/src/main/java/io/harness/beans/steps/stepinfo/DockerStepInfo.java:59`

Step env assembly entry point for Docker Registry:
- `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:325`
- `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:1326`

Plugin images configured for Docker Registry:
- Kaniko: `332-ci-manager/config/ci-manager-config.yml:380`
- Buildx: `332-ci-manager/config/ci-manager-config.yml:400`

## Phase 2: Docker Connector Auth Methods & Required Fields

Supported Docker auth types:
- `USER_PASSWORD`, `ANONYMOUS`: `954-connector-beans/src/main/java/io/harness/delegate/beans/connector/docker/DockerAuthType.java:18`

Required fields by auth method:

Username + Password:
- `dockerRegistryUrl` (required): `954-connector-beans/src/main/java/io/harness/delegate/beans/connector/DockerConnectorDTO.java:56`
- `providerType` (required): `954-connector-beans/src/main/java/io/harness/delegate/beans/connector/DockerConnectorDTO.java:57`
- `auth` (required): `954-connector-beans/src/main/java/io/harness/delegate/beans/connector/DockerConnectorDTO.java:58`
- `username` or `usernameRef` (one-of): `954-connector-beans/src/main/java/io/harness/delegate/beans/connector/docker/DockerUserNamePasswordDTO.java:27`
- `passwordRef` (required): `954-connector-beans/src/main/java/io/harness/delegate/beans/connector/docker/DockerUserNamePasswordDTO.java:34`

Anonymous:
- `dockerRegistryUrl`, `providerType`, `auth.type=ANONYMOUS` are still required.
- Auth type field is required: `954-connector-beans/src/main/java/io/harness/delegate/beans/connector/docker/DockerAuthenticationDTO.java:34`
- `getDecryptableEntities()` returns null for anonymous auth: `954-connector-beans/src/main/java/io/harness/delegate/beans/connector/DockerConnectorDTO.java:80`

## Phase 3: Connector → Plugin Env Mapping (Per Auth Flow)

### Mapping Chain Overview

Connector env var names are decided by step type:
- Docker secret env name mapping: `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:719`

The mapping is applied on both K8 and VM paths:
- K8 conversion info uses the mapping: `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/integrationstage/k8s/K8InitializeStepUtils.java:2219`
- VM serializer merges the mapping into connector details: `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/serializer/vm/VmPluginCompatibleStepSerializer.java:267`

Connector secrets are materialized here:
- Docker connector secret population (including registry URL): `933-ci-commons/src/main/java/io/harness/connector/ConnectorEnvVariablesHelper.java:443`

Base image connector mapping:
- Base image registry env mapping: `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:770`
- Base image secret env mapping: `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:793`

### Auth Flow Details

```
Auth Method: Username + Password
├── Connector Fields Used: dockerRegistryUrl, auth.type=USER_PASSWORD, credentials.username|usernameRef, passwordRef
├── Mapped to Plugin Env/Args: PLUGIN_REGISTRY, PLUGIN_USERNAME, PLUGIN_PASSW
└── Code Location: env var names `PluginSettingUtils.java:719`; values `ConnectorEnvVariablesHelper.java:443`
```

```
Auth Method: Anonymous
├── Connector Fields Used: dockerRegistryUrl, auth.type=ANONYMOUS
├── Mapped to Plugin Env/Args: PLUGIN_REGISTRY (registry URL only)
└── Code Location: auth type `DockerAuthType.java:18`; registry env set in `ConnectorEnvVariablesHelper.java:510`
```

## Phase 4: Plugin Inputs Summary (All Inputs)

Primary Docker Registry env assembly:
- `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:1326`

### Step Inputs → Plugin Env Vars

| Input Name | Type | Required | Description | Maps To (Plugin) |
|---|---|---:|---|---|
| connectorRef | string | Yes | Docker connector reference | Connector-derived envs (`PLUGIN_USERNAME`, `PLUGIN_PASSW`, `PLUGIN_REGISTRY`) via `PluginSettingUtils.java:719` + `ConnectorEnvVariablesHelper.java:443` |
| registryRef | string | No | HAR registry reference (when enabled) | Used to rewrite `repo` for HAR; no direct env var (`PluginSettingUtils.java:1331`) |
| repo | string | Yes | Image repo/name | `PLUGIN_REPO` (`PluginSettingUtils.java:1349`) |
| tags | list<string> | Yes | Image tags | `PLUGIN_TAGS` (`PluginSettingUtils.java:1350`) |
| dockerfile | string | No | Dockerfile path | `PLUGIN_DOCKERFILE` (`PluginSettingUtils.java:1354`) |
| context | string | No | Build context | `PLUGIN_CONTEXT` (`PluginSettingUtils.java:1362`) |
| target | string | No | Docker target stage | `PLUGIN_TARGET` (`PluginSettingUtils.java:1368`) |
| buildArgs | map<string,string> | No | Docker build args | `PLUGIN_BUILD_ARGS`, `PLUGIN_BUILD_ARGS_NEW` (`PluginSettingUtils.java:1374`) |
| labels | map<string,string> | No | Image labels | `PLUGIN_CUSTOM_LABELS` (`PluginSettingUtils.java:1381`) |
| baseImageConnectorRefs | list<string> | No | Base image connector(s) | `PLUGIN_BASE_IMAGE_REGISTRY`, plus base image creds envs (`PluginSettingUtils.java:1387`, `PluginSettingUtils.java:793`) |
| envDockerSecrets | map<string,string> | No | Docker secrets via env (buildx) | `PLUGIN_SECRETS_FROM_ENV` (`PluginSettingUtils.java:1463`) |
| fileDockerSecrets | map<string,string> | No | Docker secrets via file (buildx) | `PLUGIN_SECRETS_FROM_FILE` (`PluginSettingUtils.java:1463`) |
| cacheFrom | list<string> | No | Cache sources | `PLUGIN_CACHE_FROM` (`PluginSettingUtils.java:1444`, `PluginSettingUtils.java:1454`) |
| cacheTo | string | No | Cache destination | `PLUGIN_CACHE_TO` (`PluginSettingUtils.java:1394`) |
| caching | boolean | No | Enables buildx/DLC behavior | `PLUGIN_BUILDER_DRIVER_OPTS_NEW`, `PLUGIN_BUILDER_DRIVER_OPTS`, optional `PLUGIN_CACHE_METRICS_FILE` (`PluginSettingUtils.java:1400`) |
| remoteCacheRepo | string | No | Remote cache repo (K8 only) | `PLUGIN_ENABLE_CACHE` + `PLUGIN_CACHE_REPO` (non-buildx) or appended to `cacheFrom` (`PluginSettingUtils.java:1429`) |
| optimize | boolean | No | K8 snapshot optimization | `PLUGIN_SNAPSHOT_MODE=redo` on K8 (`PluginSettingUtils.java:1421`) |
| envVariables | map<string,string> | No | Extra env vars | Merged directly (`PluginSettingUtils.java:1458`) |
| runAsUser | integer | No | Container user override | VM runtime setting, not plugin env (`VmPluginCompatibleStepSerializer.java:237`) |
| resources | object | No | Container resources | Container runtime setting (`DockerStepInfo.java:74`) |
| retry | integer | No | Step retry count | Defaults to 1 (`DockerStepInfo.java:59`) |

### Auth Flow Matrix

| Auth Method | Connector Fields | Plugin Env Vars | Notes |
|---|---|---|---|
| Username + Password | `dockerRegistryUrl`, `auth.type=USER_PASSWORD`, `username|usernameRef`, `passwordRef` | `PLUGIN_REGISTRY`, `PLUGIN_USERNAME`, `PLUGIN_PASSW` | Env names: `PluginSettingUtils.java:719`; values: `ConnectorEnvVariablesHelper.java:443` |
| Anonymous | `dockerRegistryUrl`, `auth.type=ANONYMOUS` | `PLUGIN_REGISTRY` | Registry env set even without creds: `ConnectorEnvVariablesHelper.java:510` |

## Infra Applicability (K8 vs VM vs Hosted/DLITE)

Infra types for CI steps are:
- `K8`, `VM`, and `DLITE_VM`: `879-pipeline-ci-commons/src/main/java/io/harness/beans/sweepingoutputs/StageInfraDetails.java:33`

K8 flows call the plugin env builder with `Type.K8`:
- `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/serializer/PluginCompatibleStepSerializer.java:76`

VM and hosted/DLITE flows call it with `Type.VM`:
- `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/serializer/vm/VmPluginCompatibleStepSerializer.java:112`

The Docker Registry step’s env assembly explicitly branches on infra type:
- `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:1420`

### Applicability Matrix

| Concern | K8 | VM | Hosted/DLITE_VM | Notes |
|---|---:|---:|---:|---|
| Connector auth env name mapping (DOCKER_* → PLUGIN_*) | Yes | Yes | Yes (via VM path) | Mapping: `PluginSettingUtils.java:719`; K8 uses it: `K8InitializeStepUtils.java:2219`; VM merges it: `VmPluginCompatibleStepSerializer.java:267` |
| Base image connector envs and secrets | Yes | Yes | Yes (via VM path) | Base image registry envs: `PluginSettingUtils.java:770`; base image secret envs: `PluginSettingUtils.java:793` |
| Core build/push plugin envs (`PLUGIN_REPO`, `PLUGIN_TAGS`, etc.) | Yes | Yes | Yes (via VM path) | Env assembly: `PluginSettingUtils.java:1326` |
| K8-only env additions (`PLUGIN_SNAPSHOT_MODE`, `PLUGIN_ARTIFACT_FILE`, K8 cache handling) | Yes | No | No | K8 branch: `PluginSettingUtils.java:1420` |
| VM/Hosted-only env additions (`PLUGIN_DAEMON_OFF`, VM metadata file) | No | Yes | Yes | VM branch: `PluginSettingUtils.java:1447` |

## Key Takeaways

- The Docker Registry step uses the Docker connector with only two auth modes: username/password or anonymous.
- Connector credentials are mapped into `PLUGIN_USERNAME`, `PLUGIN_PASSW`, and `PLUGIN_REGISTRY`, not standard `DOCKER_*` envs.
- Build inputs are mapped into plugin envs in `PluginSettingUtils.getDockerStepInfoEnvVariables`, with K8 and VM branches for cache and snapshot behavior.
