# Git Clone: Inputs & Git Connector Mapping

This document traces the CI Manager implementation of the **Git Clone** step, its user-facing inputs, git connector auth flows, connector → plugin env mapping, and infra-type applicability.

All code locations below are workspace-relative `path:line` references.

type: GitClone
Plugin used: harness/drone-git:1.7.10-rootless (K8), github.com/wings-software/drone-git@refs/tags/v1.7.10 (VM containerless)

## Known Gaps / Gotchas

- `connectorRef` is optional on the step spec, but git clone is mandatory when enabled; missing connector is allowed only for Harness Code (Gitness) when CODE is enabled.
- Optional on spec: `879-pipeline-ci-commons/src/main/java/io/harness/beans/steps/stepinfo/GitCloneStepInfo.java:80`
- Git connector required when git clone is enabled: `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/buildstate/ConnectorUtils.java:229`
- Harness Code fallback when connector is empty: `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/buildstate/ConnectorUtils.java:244`
- `cloneDirectory` cannot be `/harness` for non-implicit GitClone steps; it throws a user exception.
- Validation and restriction: `879-pipeline-ci-commons/src/main/java/io/harness/plugin/service/PluginServiceImpl.java:322`
- On MacOS, clone directory defaults are skipped (no DRONE_WORKSPACE set unless provided).
- MacOS handling: `879-pipeline-ci-commons/src/main/java/io/harness/plugin/service/PluginServiceImpl.java:331`
- The Git Clone step uses `GIT_CLONE` and `GIT_CLONE_V1` variants; this doc focuses on `GitClone` (v0 spec).
- Switch routing: `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:389`

## Phase 1: Step Definition & Executor

Step registration and display metadata:
- `332-ci-manager/service/src/main/java/io/harness/ci/execution/plan/creator/CIPipelineServiceInfoProvider.java:281`

Step node wiring to the spec:
- `879-pipeline-ci-commons/src/main/java/io/harness/beans/steps/nodes/GitCloneStepNode.java:40`
- `879-pipeline-ci-commons/src/main/java/io/harness/beans/steps/nodes/GitCloneStepNode.java:49`

Plan creator:
- `879-pipeline-ci-commons/src/main/java/io/harness/ci/plancreator/GitCloneStepPlanCreator.java:19`

Executor (engine step):
- `332-ci-manager/service/src/main/java/io/harness/ci/execution/registrars/ExecutionRegistrar.java:89`
- `332-ci-manager/service/src/main/java/io/harness/ci/execution/states/GitCloneStep.java:16`

User-facing spec model (inputs live here):
- `879-pipeline-ci-commons/src/main/java/io/harness/beans/steps/stepinfo/GitCloneStepInfo.java:52`

Default retry constant:
- `879-pipeline-ci-commons/src/main/java/io/harness/beans/steps/stepinfo/GitCloneStepInfo.java:59`

Plugin images configured for Git Clone:
- K8 plugin image: `332-ci-manager/config/ci-manager-config.yml:372`
- VM containerless plugin: `332-ci-manager/config/ci-manager-config.yml:684`

## Phase 2: Git Connector Auth Methods & Required Fields

Git connector types supported for env mapping:
- GitHub, GitLab, Bitbucket, Azure Repos, CodeCommit, Generic Git, Harness Code.
- Connector dispatch: `879-pipeline-ci-commons/src/main/java/io/harness/utils/CiCodebaseUtils.java:128`

Auth-type validations for common providers:
- GitHub HTTP supports username/password, username/token, OAuth, GitHub App; SSH is supported.
- `879-pipeline-ci-commons/src/main/java/io/harness/utils/CiCodebaseUtils.java:188`
- GitLab HTTP supports username/password, username/token, OAuth; SSH is supported.
- `879-pipeline-ci-commons/src/main/java/io/harness/utils/CiCodebaseUtils.java:255`
- Bitbucket HTTP supports username/password and username/token; SSH is supported.
- `879-pipeline-ci-commons/src/main/java/io/harness/utils/CiCodebaseUtils.java:222`
- Azure Repos HTTP supports username/token; SSH is supported.
- `879-pipeline-ci-commons/src/main/java/io/harness/utils/CiCodebaseUtils.java:236`
- Generic Git supports HTTP or SSH.
- `879-pipeline-ci-commons/src/main/java/io/harness/utils/CiCodebaseUtils.java:281`
- CodeCommit supports HTTPS auth and uses AWS access/secret key.
- Secret handling: `933-ci-commons/src/main/java/io/harness/connector/SecretSpecBuilder.java:372`

Connector detail assembly for Git clone:
- Fetch connector (with token where applicable): `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/integrationstage/CodebaseUtils.java:404`

## Phase 3: Connector → Plugin Env Mapping (Per Auth Flow)

### Mapping Chain Overview

Git clone envs are assembled here:
- Step env assembly (GitClone v0): `879-pipeline-ci-commons/src/main/java/io/harness/plugin/service/PluginServiceImpl.java:137`

Git connector envs for remote URL, netrc machine, scm type, and optional http/ssh URL:
- `879-pipeline-ci-commons/src/main/java/io/harness/utils/CiCodebaseUtils.java:273`
- `879-pipeline-ci-commons/src/main/java/io/harness/utils/CiCodebaseUtils.java:297`

Git advanced envs (LFS/debug/fetchTags/sparse checkout/pre-fetch/submodule strategy):
- `879-pipeline-ci-commons/src/main/java/io/harness/utils/CiCodebaseUtils.java:334`

Git secrets for HTTP/SSH are materialized via SecretSpecBuilder in VM init:
- `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/integrationstage/VmInitializeTaskParamsBuilder.java:1347`
- `933-ci-commons/src/main/java/io/harness/connector/SecretSpecBuilder.java:261`

### Auth Flow Details

```
Auth Method: HTTP Username/Password or Username/Token
├── Connector Fields Used: username, passwordRef or tokenRef (provider-specific)
├── Mapped to Plugin Env/Args: DRONE_NETRC_USERNAME, DRONE_NETRC_PASSWORD
└── Code Location: GitHub example `SecretSpecBuilder.java:473`; Generic Git `SecretSpecBuilder.java:336`
```

```
Auth Method: SSH
├── Connector Fields Used: sshKeyRef (and optional passphrase)
├── Mapped to Plugin Env/Args: DRONE_SSH_KEY, DRONE_SSH_PASSPHRASE
└── Code Location: SSH secret mapping `SecretParamsUtils.java:49`; Generic Git uses `SecretSpecBuilder.java:355`
```

```
Auth Method: CodeCommit (HTTPS)
├── Connector Fields Used: accessKey or accessKeyRef, secretKeyRef
├── Mapped to Plugin Env/Args: DRONE_AWS_ACCESS_KEY, DRONE_AWS_SECRET_KEY
└── Code Location: `SecretSpecBuilder.java:391`
```

```
Auth Method: GitHub App (HTTP)
├── Connector Fields Used: appId, installationId, privateKeyRef
├── Mapped to Plugin Env/Args: DRONE_NETRC_PASSWORD (token) or HARNESS_GITHUB_APP_* (local runner)
└── Code Location: `SecretSpecBuilder.java:514`
```

## Phase 4: Plugin Inputs Summary (All Inputs)

Primary Git Clone env assembly:
- `879-pipeline-ci-commons/src/main/java/io/harness/plugin/service/PluginServiceImpl.java:137`

### Step Inputs → Plugin Env Vars

| Input Name | Type | Required | Description | Maps To (Plugin) |
|---|---|---:|---|---|
| connectorRef | string | No | Git connector reference | Connector-derived envs and secrets (see auth flow matrix) |
| repoName | string | No | Repository name | Used to build `DRONE_REMOTE_URL`, `DRONE_NETRC_MACHINE`, etc. (`CiCodebaseUtils.java:273`) |
| projectName | string | No | Project name (SCM-specific) | Not directly mapped to envs in Git clone builder |
| build | Build | Yes | Build spec (branch/tag/pr/commit) | `DRONE_COMMIT_BRANCH`, `DRONE_TAG`, `DRONE_BUILD_EVENT`, `DRONE_COMMIT_SHA` (`PluginServiceImpl.java:203`) |
| depth | integer | No | Clone depth | `PLUGIN_DEPTH` (`PluginServiceImpl.java:377`) |
| sslVerify | boolean | No | SSL verification | `GIT_SSL_NO_VERIFY` when false (`PluginServiceImpl.java:120`) |
| cloneDirectory | string | No | Clone directory | `DRONE_WORKSPACE` (`PluginServiceImpl.java:322`) |
| outputFilePathsContent | list<string> | No | Output files for content | `PLUGIN_OUTPUT_FILE_PATHS_CONTENT` (`PluginServiceImpl.java:390`) |
| lfs | boolean | No | Git LFS | `DRONE_NETRC_LFS_ENABLED` (`CiCodebaseUtils.java:334`) |
| debug | boolean | No | Debug logging | `DRONE_NETRC_DEBUG` (`CiCodebaseUtils.java:339`) |
| fetchTags | boolean | No | Fetch tags | `DRONE_NETRC_FETCH_TAGS` (`CiCodebaseUtils.java:342`) |
| sparseCheckout | list<string> | No | Sparse checkout paths | `DRONE_NETRC_SPARSE_CHECKOUT` (`CiCodebaseUtils.java:345`) |
| submoduleStrategy | enum | No | Submodule strategy | `DRONE_NETRC_SUBMODULE_STRATEGY` (`CiCodebaseUtils.java:363`) |
| preFetchCommand | string | No | Pre-fetch shell commands | `DRONE_NETRC_PRE_FETCH` (`CiCodebaseUtils.java:359`) |
| privileged | boolean | No | Privileged mode | Runtime setting, not plugin env (`GitCloneStepInfo.java:96`) |
| runAsUser | integer | No | Container user override | VM runtime setting, not plugin env (`GitCloneStepInfo.java:84`) |
| resources | object | No | Container resources | Container runtime setting (`GitCloneStepInfo.java:82`) |
| retry | integer | No | Step retry count | Uses `DEFAULT_RETRY` constant if set elsewhere (`GitCloneStepInfo.java:59`) |

Additional envs set by the builder:
- `PLUGIN_BUILD_TOOL_FILE`: `PluginServiceImpl.java:155`
- `DRONE_REMOTE_URL`, `DRONE_NETRC_MACHINE`, `DRONE_REPO_SCM`, `DRONE_GIT_HTTP_URL`, `DRONE_GIT_SSH_URL`: `CiCodebaseUtils.java:273`

### Auth Flow Matrix

| Auth Method | Connector Fields | Plugin Env Vars | Notes |
|---|---|---|---|
| HTTP (username/password or token) | username, passwordRef or tokenRef | `DRONE_NETRC_USERNAME`, `DRONE_NETRC_PASSWORD` | GitHub example `SecretSpecBuilder.java:473`; Generic Git `SecretSpecBuilder.java:336` |
| SSH | sshKeyRef (optional passphrase) | `DRONE_SSH_KEY`, `DRONE_SSH_PASSPHRASE` | SSH secret mapping `SecretParamsUtils.java:49` |
| CodeCommit HTTPS | accessKey/accessKeyRef, secretKeyRef | `DRONE_AWS_ACCESS_KEY`, `DRONE_AWS_SECRET_KEY`, `DRONE_AWS_REGION` | Secrets: `SecretSpecBuilder.java:391`; Region: `CiCodebaseUtils.java:370` |
| GitHub App | appId, installationId, privateKeyRef | `DRONE_NETRC_PASSWORD` or `HARNESS_GITHUB_APP_*` | `SecretSpecBuilder.java:514` |

## Infra Applicability (K8 vs VM vs Hosted/DLITE)

Infra types for CI steps are:
- `K8`, `VM`, and `DLITE_VM`: `879-pipeline-ci-commons/src/main/java/io/harness/beans/sweepingoutputs/StageInfraDetails.java:33`

K8 flows call the plugin env builder via PluginSettingUtils:
- `877-pipeline-ci-cd-commons/src/java/io/harness/iacm/execution/PluginSettingUtils.java:396`

VM and hosted/DLITE flows use a dedicated Git clone serializer:
- `879-pipeline-ci-commons/src/main/java/io/harness/plugin/service/VMGitCloneStepSerializer.java:64`

### Applicability Matrix

| Concern | K8 | VM | Hosted/DLITE_VM | Notes |
|---|---:|---:|---:|---|
| Git env assembly (`DRONE_*`, `PLUGIN_*`) | Yes | Yes | Yes (via VM path) | K8 uses PluginSettingUtils (`PluginSettingUtils.java:396`); VM uses `VMGitCloneStepSerializer.java:72` |
| Connector secrets injection (netrc/ssh keys) | Yes | Yes | Yes | VM injects decrypted git secrets: `VmInitializeTaskParamsBuilder.java:1347`; K8 uses connector details and secret refs in init |
| Clone directory handling (`DRONE_WORKSPACE`) | Yes | Yes | Yes | `PluginServiceImpl.java:322` |

## Key Takeaways

- Git Clone builds DRONE-style env variables for the drone-git plugin and uses Git connector details to generate remote URLs and auth envs.
- Credentials are exposed as DRONE_NETRC_* or DRONE_SSH_* envs depending on HTTP vs SSH auth, with CodeCommit mapped to AWS access/secret keys.
- Both K8 and VM paths use the same env assembly logic, with VM injecting decrypted git secrets explicitly during initialization.

## Harness Code Repository (Connector-less) Option

This section explains how the **Harness Code Repository** option works for Git Clone when **no connector is provided** and the user supplies a repository identifier (account/org/project scoped). In code, this is represented by a **blank `connectorRef`** plus a populated `repoName`.

### How the Step Detects Harness Code

When `connectorRef` is blank, the Git Clone step is tagged as a Harness Code operation and the repo identifier is carried forward:
- `StepOperationMetadata.isHarnessCode=true` when `connectorRef` is blank: `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/integrationstage/k8s/K8InitializeStepUtils.java:1186`
- The repo identifier is taken from `repoName`: `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/integrationstage/k8s/K8InitializeStepUtils.java:1189`

Plan creation also **intentionally allows** a blank connector when a repo name is present so downstream Harness Code logic can kick in:
- Inline pipeline logic allows `connector` blank if `repository.name` is set: `332-ci-manager/service/src/main/java/io/harness/ci/execution/integrationstage/V1/CIPlanCreatorUtils.java:559`

### How Connector Resolution Works Without a Connector

K8 initialization reads the Harness Code metadata and passes the repo identifier to connector resolution:
- Passes `harnessCodeRepo` into connector resolution: `332-ci-manager/service/src/main/java/io/harness/ci/execution/integrationstage/K8InitializeTaskParamsBuilder.java:593`
- Calls `codebaseUtils.getGitConnector(..., harnessCodeRepo)`: `332-ci-manager/service/src/main/java/io/harness/ci/execution/integrationstage/K8InitializeTaskParamsBuilder.java:616`

Connector resolution builds a **Harness Code connector** with a JWT token when `connectorRef` is empty but `repoName` is present:
- Empty connector + repoName triggers Harness Code path: `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/buildstate/ConnectorUtils.java:244`
- Token is minted with repo-scoped claims: `877-pipeline-ci-cd-commons/src/java/io/harness/ci/execution/buildstate/ConnectorUtils.java:274`
- Token uses the repo slug as an allowed resource: `954-connector-beans/src/main/java/io/harness/connector/utils/HarnessCodeConnectorUtils.java:85`

### How the Repo URL Is Built

The Harness Code connector translates the repo identifier into a full git URL:
- Harness connector uses repo slug (account/org/project/repo) in the URL: `933-ci-commons/src/main/java/io/harness/connector/CiIntegrationStageUtils.java:137`
- Identifier resolution handles account/org/project scoping: `933-ci-commons/src/main/java/io/harness/connector/CiIntegrationStageUtils.java:144`

### Mapping Summary (Harness Code Option)

| User Input | Stored In | Used By | Outcome |
|---|---|---|---|
| `repoName` (scoped repo identifier) | `GitCloneStepInfo.repoName` | Step metadata + connector utils | JWT token scoped to repo; repo URL constructed from account/org/project/repo |
| `connectorRef` omitted | `GitCloneStepInfo.connectorRef` blank | Harness Code detection | Harness Code connector path is used |

### UI Field / Input

| Field / Input (UI) | Visible In Build Flow(s) | Optional? | Tooltip ID | Tooltip Text / Content | Feature-Flag Influence | License Influence |
|---|---|---|---|---|---|---|
| `identifier` | All | No | — | — | None | None |
| `name` | All | No | — | — | None | None |
| `description` | All | Yes | — | — | None | None |
| `provider` (Git provider selector) | All (only when FF enabled) | Yes (UI selector) | — | — | `CODE_ENABLED`: when off, provider selector is hidden and flow defaults to non-Harness-Code path | None |
| `spec.connectorRef` (inside connector+repo control) | All (when non-Harness-Code provider) | Conditional | `rightBarForm_connectorRef` | “Use the dropdown to select an existing Harness Git connector or to create a new connector. [Learn More](https://docs.harness.io/article/v9sigwjlgo)” | `CODE_ENABLED` affects whether Harness Code provider can hide connector requirement | None |
| `spec.repoName` (inside connector+repo control) | All | Conditional | `rightBarForm_repoName` | “If URL Type is Git Repository, repo name is auto shown. If URL Type is Git Account, provide repo name at pipeline use time.” | `CODE_ENABLED` affects provider path and repo behavior | None |
| `spec.build.type` | All | No | `buildType` | Select Git Branch / Git Tag (or Commit option in UI radio group), then provide corresponding ref. | None | None |
| `spec.build.spec.branch` | All (only when build type = `branch`) | Conditional (by build type) | — | — | None | None |
| `spec.build.spec.tag` | All (only when build type = `tag`) | Conditional (by build type) | — | — | None | None |
| `spec.build.spec.commitSha` | All (only when build type = `commitSha`) | Conditional (by build type) | — | — | None | None |
| `spec.cloneDirectory` | All | Yes | `cloneDirectory` | “Enter an optional target path in the stage workspace where you want to clone the repo.” | None | None |
| `spec.depth` | All | Yes | `depth` | “Number of commits to fetch when cloning. Default 50 for manual; 0 for other triggers.” | None | None |
| `spec.prCloneStrategy` | All | Yes | `prCloneStrategy` | Controls PR build artifact source: merge commit vs source branch. | None | None |
| `spec.fetchTags` | All | Yes | `cloneCodebase_fetchTags` | “Fetch all tags for shallow clone (`depth > 0`), equivalent to `--tags`.” | None | None |
| `spec.lfs` | All | Yes | `cloneCodebase_lfs` | “Set true to download Git-LFS files.” | None | None |
| `spec.sparseCheckout` | All | Yes | `cloneCodebase_sparseCheckout` | “Do sparse checkout on given patterns (cone mode directories).” | None | None |
| `spec.submoduleStrategy` | All | Yes | `cloneCodebase_submoduleStrategy` | “Include submodules (true/recursive).” | None | None |
| `spec.preFetchCommand` | All | Yes | `cloneCodebase_prefetchCommand` | “Additional Git commands before fetch; one command per line.” | None | None |
| `spec.sslVerify` | All | Yes | `sslVerify` | “If true (default), verify Git SSL certs; false only for known cert issues.” | None | None |
| `spec.privileged` | `Cloud`, `KubernetesDirect` only | Yes | `privileged` | Run containers with Docker `--privileged` (elevated privileges/security risk). | None | None |
| `spec.runAsUser` | All (forced visible for this step) | Yes | `runAsUser` | “Specify UID to run processes in pod/containers.” | None | None |
| `spec.limitMemory` | `KubernetesDirect`, `KubernetesHosted` only | Yes | `limitMemory` | “Maximum memory for container (default 500Mi).” | None | None |
| `spec.limitCPU` | `KubernetesDirect`, `KubernetesHosted` only | Yes | `limitCPULabel` | “Maximum CPU for container (default 400m).” | None | None |
| `timeout` | All | Yes | — (edit view) | — | None | None |

Conditional requiredness in this step:
- `spec.connectorRef` is required when provider is non-Harness-Code.
- `spec.repoName` is required for Harness-Code provider, or for Account-type connector URL mode.

### Notes

- The Harness Code path is gated by `CODE_ENABLED` and uses the configured `harnessCodeGitBaseUrl` (see `ConnectorUtils.java:214` + `332-ci-manager/config/ci-manager-config.yml:725` for the config key).
- With this option, **no SCM connector is required**; authentication is handled via a service-issued JWT scoped to the repo identifier.
