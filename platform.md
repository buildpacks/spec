# Platform Interface Specification (DEPRECATED)

## This API version is considered [deprecated](https://github.com/buildpacks/rfcs/blob/main/text/0049-multi-api-lifecycle-descriptor.md) and will become unsupported as of July 1, 2023.

For information about how to migrate, consult the [migration guides](https://buildpacks.io/docs/reference/spec/migration/).

This document specifies the interface between a lifecycle and a platform.

A platform orchestrates a lifecycle to make buildpack functionality available to end-users such as application developers.

Examples of a platform might include:

1. A local CLI tool that uses buildpacks to create OCI images
1. A plugin for a continuous integration service that uses buildpacks to create OCI images
1. A cloud application platform that uses buildpacks to build source code before deployment

## Table of Contents

<!-- Using https://github.com/yzhang-gh/vscode-markdown to manage toc -->
- [Platform Interface Specification](#platform-interface-specification)
  - [Table of Contents](#table-of-contents)
  - [Platform API Version](#platform-api-version)
  - [Terminology](#terminology)
      - [CNB Terminology](#cnb-terminology)
      - [Additional Terminology](#additional-terminology)
  - [Stacks](#stacks)
    - [Stack ID](#stack-id)
    - [Build Image](#build-image)
    - [Run Image](#run-image)
    - [Mixins](#mixins)
    - [Compatibility Guarantees](#compatibility-guarantees)
  - [Lifecycle Interface](#lifecycle-interface)
    - [Platform API Compatibility](#platform-api-compatibility)
    - [Operations](#operations)
      - [Build](#build)
      - [Rebase](#rebase)
      - [Launch](#launch)
    - [Usage](#usage)
      - [`detector`](#detector)
        - [Inputs](#inputs)
        - [Outputs](#outputs)
      - [`analyzer`](#analyzer)
        - [Inputs](#inputs-1)
        - [Outputs](#outputs-1)
        - [Layer analysis](#layer-analysis)
      - [`restorer`](#restorer)
        - [Inputs](#inputs-2)
        - [Outputs](#outputs-2)
        - [Layer restoration](#layer-restoration)
      - [`builder`](#builder)
        - [Inputs](#inputs-3)
        - [Outputs](#outputs-3)
      - [`exporter`](#exporter)
        - [Inputs](#inputs-4)
        - [Outputs](#outputs-4)
      - [`creator`](#creator)
        - [Inputs](#inputs-5)
        - [Outputs](#outputs-5)
      - [`rebaser`](#rebaser)
        - [Inputs](#inputs-6)
        - [Outputs](#outputs-6)
      - [`launcher`](#launcher)
        - [Inputs](#inputs-7)
        - [Outputs](#outputs-7)
    - [Run Image Resolution](#run-image-resolution)
    - [Registry Authentication](#registry-authentication)
  - [Buildpacks](#buildpacks)
    - [Buildpacks Directory Layout](#buildpacks-directory-layout)
  - [Security Considerations](#security-considerations)
  - [Additional Guidance](#additional-guidance)
    - [Environment](#environment)
      - [Buildpack Environment](#buildpack-environment)
        - [Stack-Provided Variables](#stack-provided-variables)
        - [User-Provided Variables](#user-provided-variables)
      - [Launch Environment](#launch-environment)
    - [Caching](#caching)
    - [Build Reproducibility](#build-reproducibility)
  - [Data Format](#data-format)
    - [Files](#files)
      - [`analyzed.toml` (TOML)](#analyzedtoml-toml)
      - [`group.toml` (TOML)](#grouptoml-toml)
      - [`metadata.toml` (TOML)](#metadatatoml-toml)
      - [`order.toml` (TOML)](#ordertoml-toml)
      - [`plan.toml` (TOML)](#plantoml-toml)
      - [`project-metadata.toml` (TOML)](#project-metadatatoml-toml)
      - [`report.toml` (TOML)](#reporttoml-toml)
      - [`stack.toml` (TOML)](#stacktoml-toml)
    - [Labels](#labels)
      - [`io.buildpacks.build.metadata` (JSON)](#iobuildpacksbuildmetadata-json)
      - [`io.buildpacks.lifecycle.metadata` (JSON)](#iobuildpackslifecyclemetadata-json)
      - [`io.buildpacks.project.metadata` (JSON)](#iobuildpacksprojectmetadata-json)

## Platform API Version

This document specifies Platform API version `0.5`.

Platform API versions:
 - MUST be in form `<major>.<minor>` or `<major>`, where `<major>` is equivalent to `<major>.0`
 - When `<major>` is greater than `0` increments to `<minor>` SHALL exclusively indicate additive changes

## Terminology

#### CNB Terminology
A **buildpack** refers to software compliant with the [Buildpack Interface Specification](buildpack.md).

A **stack** is a contract, implemented by a **build image** and **run image**, that guarantees properties of the **build environment** and **app image**.

A **stack ID** uniquely identifies a particular **stack**.

A **build image** is an OCI image that provides the base of the **build environment**.

A **run image** is an OCI image that provides the base from which **app images** are built.

A **mixin** is a named set of additions to a stack that can be used to make additive changes to the contract.

The **build environment** refers to the containerized environment in which the lifecycle executes buildpacks.

An **app image** refers to an OCI image generated by the lifecycle by extending the **run image** with any or all of the following: **app layers**, **launch layers**, **launcher layers**, image configuration.

A **launch layer** refers to a layer in the app image created from a  `<layers>/<layer>` directory as specified in the [Buildpack Interface Specification](buildpack.md).

An **app layer** refers to a layer in the app image created from the `<app>` directory as specified in the [Buildpack Interface Specification](buildpack.md).

A **run image layer** refers to a layer in the **app image** originating from the **run image**.

A **launcher layer** refers to a layer in the app OCI image containing the **launcher** itself and/or launcher configuration.

The **launcher** refers to a lifecycle executable packaged in the **app image** for the purpose of executing processes at runtime.

#### Additional Terminology
An **image reference** refers to either a **tag reference** or **digest reference**.

A **tag reference** refers to an identifier of form `<registry>/<repo>:<tag>` which locates an image manifest in an [OCI Distribution Specification](https://github.com/opencontainers/distribution-spec/blob/master/spec.md) compliant registry.

A **digest reference**  refers to a [content addressable](https://en.wikipedia.org/wiki/Content-addressable_storage) identifier of form `<registry>/<repo>@<digest>` which locates an image manifest in an [OCI Distribution Specification](https://github.com/opencontainers/distribution-spec/blob/master/spec.md) compliant registry.

The following is a non-exhaustive list of terms defined in the [OCI Image Format Specification](https://github.com/opencontainers/image-spec) used throughout this document:
* **image config** https://github.com/opencontainers/image-spec/blob/master/config.md#oci-image-configuration
* **imageID** - https://github.com/opencontainers/image-spec/blob/master/config.md#imageid
* **diffID** - https://github.com/opencontainers/image-spec/blob/master/config.md#layer-diffid

## Stacks

A typical stack might specify:
* The OS distro in the build environment.
* OS packages installed in the build environment.
* Trusted CA certificates in the build environment.
* The OS distro or distroless OS in the launch environment.
* OS packages installed in the launch environment.
* Trusted CA certificates in the launch environment.
* The default user and the build and launch environments.

Stack authors SHOULD define the contract such that any stack images CVEs can be addressed with security patches without violating the [compatibility guarantees](#compatibility-guarantees).

### Stack ID

Stack authors MUST choose a globally unique ID, for example: "io.buildpacks.mystack".

The stack ID:
- MUST NOT be identical to any other stack ID when using a case-insensitive comparison.
- MUST only contain numbers, letters, and the characters `.`, `/`, and `-`.
- SHOULD use reverse domain name notation to avoid name collisions.

### Build Image

The platform MUST ensure that:

- The image config's `User` field is set to a non-root user with a writable home directory.
- The image config's `Env` field has the environment variable `CNB_STACK_ID` set to the stack ID.
- The image config's `Env` field has the environment variable `CNB_USER_ID` set to the user [†](README.md#operating-system-conventions)UID/[‡](README.md#operating-system-conventions)SID of the user specified in the `User` field.
- The image config's `Env` field has the environment variable `CNB_GROUP_ID` set to the primary group [†](README.md#operating-system-conventions)GID/[‡](README.md#operating-system-conventions)SID of the user specified in the `User` field.
- The image config's `Env` field has the environment variable `PATH` set to a valid set of paths or explicitly set to empty (`PATH=`).
- The image config's `Label` field has the label `io.buildpacks.stack.id` set to the stack ID.
- The image config's `Label` field has the label `io.buildpacks.stack.mixins` set to a JSON array containing mixin names for each mixin applied to the image.

The platform SHOULD ensure that:

- The image config's `Label` field has the label `io.buildpacks.stack.maintainer` set to the name of the stack maintainer.
- The image config's `Label` field has the label `io.buildpacks.stack.homepage` set to the homepage of the stack.
- The image config's `Label` field has the label `io.buildpacks.stack.distro.name` set to the name of the stack's OS distro.
- The image config's `Label` field has the label `io.buildpacks.stack.distro.version` set to the version of the stack's OS distro.
- The image config's `Label` field has the label `io.buildpacks.stack.released` set to the release date of the stack.
- The image config's `Label` field has the label `io.buildpacks.stack.description` set to the description of the stack.
- The image config's `Label` field has the label `io.buildpacks.stack.metadata` set to additional metadata related to the stack.   

### Run Image

The platform MUST ensure that:

- The image config's `User` field is set to a user with the same user [†](README.md#operating-system-conventions)UID/[‡](README.md#operating-system-conventions)SID and primary group [†](README.md#operating-system-conventions)GID/[‡](README.md#operating-system-conventions)SID as in the build image.
- The image config's `Label` field has the label `io.buildpacks.stack.id` set to the stack ID.
- The image config's `Label` field has the label `io.buildpacks.stack.mixins` set to a JSON array containing mixin names for each mixin applied to the image.
- The image config's `Env` field has the environment variable `PATH` set to a valid set of paths or explicitly set to empty (`PATH=`).

The platform SHOULD ensure that:

- The image config's `Label` field has the label `io.buildpacks.stack.maintainer` set to the name of the stack maintainer.
- The image config's `Label` field has the label `io.buildpacks.stack.homepage` set to the homepage of the stack.
- The image config's `Label` field has the label `io.buildpacks.stack.distro.name` set to the name of the stack's OS distro.
- The image config's `Label` field has the label `io.buildpacks.stack.distro.version` set to the version of the stack's OS distro.
- The image config's `Label` field has the label `io.buildpacks.stack.released` set to the release date of the stack.
- The image config's `Label` field has the label `io.buildpacks.stack.description` set to the description of the stack.
- The image config's `Label` field has the label `io.buildpacks.stack.metadata` set to additional metadata related to the stack.   

### Mixins

A mixin name MUST only be defined by the author of its corresponding stack.
A mixin name MUST always be used to specify the same set of changes.
A mixin name MUST only contain a `:` character as part of an optional stage specifier.

A mixin prefixed with the `build:` stage specifier only affects the build image and does not need to be specified on the run image.
A mixin prefixed with the `run:` stage specifier only affects the run image and does not need to be specified on the build image.

A platform MAY support any number of mixins for a given stack in order to support application code or buildpacks that require those mixins.

Changes introduced by mixins SHOULD be restricted to the addition of operating system software packages that are regularly patched with strictly backwards-compatible security fixes.
However, mixins MAY consist of any changes that follow the [Compatibility Guarantees](#compatibility-guarantees).

### Compatibility Guarantees

Stack image authors SHOULD ensure that build image versions maintain [ABI-compatibility](https://en.wikipedia.org/wiki/Application_binary_interface) with previous versions, although violating this requirement will not change the behavior of previously built images containing app and launch layers.

Stack image authors MUST ensure that new run image versions maintain [ABI-compatibility](https://en.wikipedia.org/wiki/Application_binary_interface) with previous versions.
Stack image authors MUST ensure that app and launch layers do not change behavior when the run image layers are upgraded to newer versions, unless those behavior changes are intended to fix security vulnerabilities.

Mixin authors MUST ensure that mixins do not affect the [ABI-compatibility](https://en.wikipedia.org/wiki/Application_binary_interface) of any object code compiled to run on the base stack images without mixins.

During build, platforms MUST use the same set of mixins for the run image as were used in the build image (excluding mixins that have a stage specifier).

## Lifecycle Interface
### Platform API Compatibility

The platform SHOULD set `CNB_PLATFORM_API=<platform API version>` in the lifecycle's execution environment

If `CNB_PLATFORM_API` is set in the lifecycle's execution environment, the lifecycle:
  - MUST either conform to the matching version of this specification or
  - MUST fail if it does not support `<platform API version>`

### Operations

#### Build
A single app image build* consists of the following phases:

1. Detection
1. Analysis
1. Cache Restoration
1. Build*
1. Export

A platform MUST execute these phases either by invoking the following phase-specific lifecycle binaries in order:
1. `/cnb/lifecycle/detector`
1. `/cnb/lifecycle/analyzer`
1. `/cnb/lifecycle/restorer`
1. `/cnb/lifecycle/builder`
1. `/cnb/lifecycle/exporter`

or by executing `/cnb/lifecycle/creator`.

> \* **build** is an overloaded term that refers to both a single phase and the operation comprised of the above phases.
> The meaning of any particular instance of the word **build** must be assessed in context

#### Rebase
When an updated run image with the same stack ID is available, an updated app image SHOULD be generated from the existing app image config by replacing the run image layers in the existing app image with the layers from the new run image.
This is referred to as rebasing the app, launch, and launcher layers onto the new run image layers.
When layers are rebased, any app image metadata referenceing to the original run image MUST be updated to reference to the new run image.
This entire operation is referred to as rebasing the app image.

Rebasing allows for fast runtime OS-level dependency updates for app images without requiring a rebuild. A rebase requires minimal data transfer when the app and run images are colocated on a Docker registry that supports [Cross Repository Blob Mounts](https://docs.docker.com/registry/spec/api/#cross-repository-blob-mount).

To rebase an app image a platform MUST execute the `/cnb/lifecycle/rebaser` or perform an equivalent operation.
 
#### Launch
`/cnb/lifecycle/launcher` is responsible for launching user and buildpack provided processes in the correct execution environment.
`/cnb/lifecycle/launcher` SHALL be the `ENTRYPOINT` for all app images.

### Usage

All lifecycle phases:

- MUST read `CNB_PLATFORM_API` from the execution environment and evaluate compatibility before attempting to parse other inputs (see [Platform API Compatibility](#platform-api-compatibility))
- MUST give command line inputs precedence over other inputs

#### `detector`
The platform MUST execute `detector` in the **build environment**

Usage: 
```
/cnb/lifecycle/detector \
  [-app <app>] \
  [-buildpacks <buildpacks>] \
  [-group <group>] \
  [-layers <layers>] \
  [-log-level <log-level>] \
  [-order <order>] \
  [-plan <plan>] \
  [-platform <platform>]
```

##### Inputs
| Input         | Environment Variable    | Default Value             | Description
|---------------|-------------------------|---------------------------|----------------------
| `<app>`         | `CNB_APP_DIR`         | `/workspace`          | Path to application directory
| `<buildpacks>`  | `CNB_BUILDPACKS_DIR`  | `/cnb/buildpacks`     | Path to buildpacks directory (see [Buildpacks Directory Layout](#buildpacks-directory-layout))
| `<group>`       | `CNB_GROUP_PATH`      | `<layers>/group.toml` | Path to output group definition
| `<layers>`      | `CNB_LAYERS_DIR`      | `/layers`             | Path to layers directory
| `<log-level>`   | `CNB_LOG_LEVEL`       | `info`                | Log Level
| `<order>`       | `CNB_ORDER_PATH`      | `/cnb/order.toml`     | Path to order definition (see [`order.toml`](#ordertoml-toml))
| `<plan>`        | `CNB_PLAN_PATH`       | `<layers>/plan.toml`  | Path to output resolved build plan
| `<platform>`    | `CNB_PLATFORM_DIR`    | `/platform`           | Path to platform directory

##### Outputs
| Output             | Description
|--------------------|----------------------------------------------
| [exit status]      | (see Exit Code table below for values)
| `/dev/stdout`      | Logs (info)
| `/dev/stderr`      | Logs (warnings, errors)
| `<group>`          | Detected buildpack group  (see [`group.toml`](#grouptoml-toml))
| `<plan>`           | Resolved Build Plan (see [`plan.toml`](#plantoml-toml))

| Exit Code | Result|
|-----------|-------|
| `0`       | Success
| `11`      | Platform API incompatibility error
| `12`      | Buildpack API incompatibility error
| `1-10`, `13-99` | Generic lifecycle errors
| `100`     | All buildpacks groups have failed to detect w/o error
| `101`     | All buildpack groups have failed to detect and at least one buildpack has errored
| `102-199` | Detection-specific lifecycle errors

The lifecycle:
- SHALL detect a single group from `<order>` and write it to `<group>` using the [detection process](buildpack.md#phase-1-detection) outlined in the Buildpack Interface Specification
- SHALL write the resolved build plan from the detected group to `<plan>`

#### `analyzer`
Usage: 
```
/cnb/lifecycle/analyzer \
  [-analyzed <analyzed>] \
  [-cache-dir <cache-dir>] \
  [-cache-image <cache-image>] \
  [-daemon] \ # sets <daemon>
  [-gid <gid>] \
  [-group <group>] \
  [-layers <layers>] \
  [-log-level <log-level>] \
  [-skip-layers <skip-layers>] \
  [-uid <uid>] \
  <image>
```

##### Inputs
| Input          | Environment Variable  | Default Value            | Description
|----------------|-----------------------|--------------------------|----------------------
| `<analyzed>`   | `CNB_ANALYZED_PATH`   | `<layers>/analyzed.toml` | Path to output analysis metadata (see [`analyzed.toml`](#analyzedtoml-toml)
| `<cache-dir>`  | `CNB_CACHE_DIR`       |                          | Location of cache, provided as a directory
| `<cache-image>`| `CNB_CACHE_IMAGE`     |                          | Location of cache, provided as an image
| `<daemon>`     | `CNB_USE_DAEMON`      | `false`                  | Analyze image from docker daemon
| `<gid>`        | `CNB_GROUP_ID`        |                          | Primary GID of the stack `User`
| `<group>`      | `CNB_GROUP_PATH`      | `<layers>/group.toml`    | Path to group definition (see [`group.toml`](#grouptoml-toml))
| `<image>`      |                       |                          | Image reference to be analyzed (usually the result of the previous build)
| `<layers>`     | `CNB_LAYERS_DIR`      | `/layers`                | Path to layers directory
| `<log-level>`  | `CNB_LOG_LEVEL`       | `info`                   | Log Level
| `<skip-layers>`| `CNB_SKIP_LAYERS`     | `false`                  | Do not perform layer analysis
| `<uid>`        | `CNB_USER_ID`         |                          | UID of the stack `User`

- **If** `<daemon>` is `false`, `<image>` MUST be a valid image reference
- **If** `<daemon>` is `true`, `<image>` MUST be either a valid image reference or an imageID
- The lifecycle MUST accept valid references to non-existent images without error.

##### Outputs
| Output             | Description
|--------------------|----------------------------------------------
| [exit status]      | (see Exit Code table below for values)
| `/dev/stdout`      | Logs (info)
| `/dev/stderr`      | Logs (warnings, errors)
| `<analyzed>`       | Analysis metadata (see [`analyzed.toml`](#analyzedtoml-toml)
| `<layers>/<buidpack-id>/<layer>.sha`  | Files containing the diffID of each analyzed layer
| `<layers>/<buidpack-id>/<layer>.toml` | Files containing the layer content metadata of each analyzed layer (see data format in [Buildpack Interface Specification](buildpack.md))

| Exit Code | Result|
|-----------|-------|
| `0`       | Success
| `11`      | Platform API incompatibility error
| `12`      | Buildpack API incompatibility error
| `1-10`, `13-99` | Generic lifecycle errors
| `200-299` | Analysis-specific lifecycle errors

- The lifecycle MUST write [analysis metadata](#analyzedtoml-toml) to `<analyzed>` if `<image>` is accessible.
- **If** `<skip-layers>` is `true` the lifecycle MUST NOT perform layer analysis.
- **Else** the lifecycle MUST analyze any app image layers or cached layers created by any buildpack present in the provided `<group>`.

##### Layer analysis
When analyzing a given layer the lifecycle SHALL:
- **If** `build=true`, `cache=false`:
    - Do nothing
- **Else if** `launch=true`:
    - Write layer metadata read from the analyzed image to `<layers>/<buildpack-id>/<layer-name>.toml`
    - Write the layer diffID from the analyzed image to `<layers>/<buildpack-id>/<layer-name>.sha`
- **Else if** `cache=true`:
    - Write layer metadata read from the cache to `<layers>/<buildpack-id>/<layer-name>.toml`
    - Write the layer diffID from the cache to `<layers>/<buildpack-id>/<layer-name>.sha`

#### `restorer`
Usage:
```
/cnb/lifecycle/restorer \
  [-cache-dir <cache-dir>]
  [-cache-image <cache-image>]
  [-gid <gid>] \
  [-group <group>] \
  [-layers <layers>] \
  [-log-level <log-level>] \
  [-uid <uid>]
```

##### Inputs
| Input          | Environment Variable  | Default Value         | Description
|----------------|-----------------------|-----------------------|----------------------
| `<cache-dir>`  | `CNB_CACHE_DIR`       |                       | Path to a cache directory
| `<cache-image>`| `CNB_CACHE_IMAGE`     |                       | Reference to a cache image in an OCI image registry
| `<gid>`        | `CNB_GROUP_ID`        |                       | Primary GID of the stack `User`
| `<group>`      | `CNB_GROUP_PATH`      | `<layers>/group.toml` | Path to group definition (see [`group.toml`](#grouptoml-toml))
| `<layers>`     | `CNB_LAYERS_DIR`      | `/layers`             | Path to layers directory
| `<log-level>`  | `CNB_LOG_LEVEL`       | `info`                | Log Level
| `<uid>`        | `CNB_USER_ID`         |                       | UID of the stack `User`
| `<layers>/<buidpack-id>/<layer>.sha`  ||                       | Files containing the diffID of each analyzed layer
| `<layers>/<buidpack-id>/<layer>.toml` ||                       | Files containing the layer content metadata of each analyzed layer (see data format in [Buildpack Interface Specification](buildpack.md))

##### Outputs
| Output                             | Description
|------------------------------------|----------------------------------------------
| [exit status]                      | (see Exit Code table below for values)
| `/dev/stdout`                      | Logs (info)
| `/dev/stderr`                      | Logs (warnings, errors)
| `<layers>/<buidpack-id>/<layer>/*` | Restored layer contents

| Exit Code | Result|
|-----------|-------|
| `0`       | Success
| `11`      | Platform API incompatibility error
| `12`      | Buildpack API incompatibility error
| `1-10`, `13-99` | Generic lifecycle errors
| `300-399` | Restoration-specific lifecycle errors

##### Layer restoration
For each layer metadata file found in the `<layers>` directory, the lifecycle:
- MUST restore cached layer contents if the cache contains a layer with matching diffID
- MUST remove layer metadata if `cache=true` AND the cache DOES NOT contain a layer with matching diffID

#### `builder`
The platform MUST execute `builder` in the **build environment**

Usage: 
```
/cnb/lifecycle/builder \
  [-app <app>] \
  [-buildpacks <buildpacks>] \
  [-group <group>] \
  [-layers <layers>] \
  [-log-level <log-level>] \
  [-plan <plan>] \
  [-platform <platform>]
```

##### Inputs
| Input          | Env                   | Default Value         | Description
|----------------|-----------------------|-----------------------|----------------------
| `<app>`        | `CNB_APP_DIR`         | `/workspace`          | Path to application directory
| `<buildpacks>` | `CNB_BUILDPACKS_DIR`  | `/cnb/buildpacks`     | Path to buildpacks directory (see [Buildpacks Directory Layout](#buildpacks-directory-layout))
| `<group>`      | `CNB_GROUP_PATH`      | `<layers>/group.toml` | Path to group definition (see [`group.toml`](#grouptoml-toml))
| `<layers>`     | `CNB_LAYERS_DIR`      | `/layers`             | Path to layers directory
| `<log-level>`  | `CNB_LOG_LEVEL`       | `info`                | Log Level
| `<plan>`       | `CNB_PLAN_PATH`       | `<layers>/plan.toml`  | Path to resolved build plan (see [`plan.toml`](#plantoml-toml))
| `<platform>`   | `CNB_PLATFORM_DIR`    | `/platform`           | Path to platform directory

##### Outputs
| Output                                     | Description
|--------------------------------------------|----------------------------------------------
| [exit status]                              | (see Exit Code table below for values)
| `/dev/stdout`                              | Logs (info)
| `/dev/stderr`                              | Logs (warnings, errors)
| `<layers>/<buildpack ID>/<layer>`          | Layer contents (see [Buildpack Interface Specfication](buildpack.md)
| `<layers>/<buildpack ID>/<layer>.toml`     | Layer metadata (see [Buildpack Interface Specfication](buildpack.md)
| `<layers>/config/metadata.toml`            | Build metadata (see [`metadata.toml`](#metadatatoml-toml))

| Exit Code | Result|
|-----------|-------|
| `0`       | Success
| `11`      | Platform API incompatibility error
| `12`      | Buildpack API incompatibility error
| `1-10`, `13-99` | Generic lifecycle errors
| `401`     | Buildpack build error
| `400`, `402-499`|  Build-specific lifecycle errors

- The lifecycle SHALL execute all buildpacks in the order defined in `<group>` according process outlined in the [Buildpack Interface Specification](buildpack.md).
- The lifecycle SHALL add all invoked buildpacks to`<layers>/config/metadata.toml`.
- The lifecycle SHALL aggregate all `processes`, `slices` and BOM entries returned by buildpacks in `<layers>/config/metadata.toml`.

#### `exporter`
Usage:
```
/cnb/lifecycle/exporter \
  [-analyzed <analyzed>] \
  [-app <app>] \
  [-cache-dir <cache-dir>] \
  [-cache-image <cache-image>] \
  [-daemon] \ # sets <daemon>
  [-gid <gid>] \
  [-group <group>] \
  [-launch-cache <launch-cache> ] \
  [-launcher <launcher> ] \
  [-layers <layers>] \
  [-log-level <log-level>] \
  [-process-type <process-type> ] \
  [-project-metadata <project-metadata> ] \
  [-report <report> ] \
  [-run-image <run-image> | -image <run-image> ] \ # -image is Deprecated
  [-stack <stack>] \
  [-uid <uid> ] \
  <image> [<image>...]
```

##### Inputs
| Input               | Environment Variable       | Default Value       | Description
|---------------------|----------------------------|---------------------|---------------------------------------
| `<analyzed>`        | `CNB_ANALYZED_PATH`        | `<layers>/analyzed.toml`  | Path to analysis metadata (see [`analyzed.toml`](#analyzedtoml-toml)
| `<app>`             | `CNB_APP_DIR`              | `/workspace`        | Path to application directory
| `<cache-dir>`       | `CNB_CACHE_DIR`            |                     | Path to a cache directory
| `<cache-image>`     | `CNB_CACHE_IMAGE`          |                     | Reference to a cache image in an OCI image registry
| `<daemon>`          | `CNB_USE_DAEMON`           | `false`             | Export image to docker daemon
| `<gid>`             | `CNB_GROUP_ID`             |                     | Primary GID of the stack `User`
| `<group>`           | `CNB_GROUP_PATH`           | `<layers>/group.toml`     | Path to group file (see [`group.toml`](#grouptoml-toml))
| `<image>`           |                            |                     | Tag reference to which the app image will be written
| `<launch-cache>`    | `CNB_LAUNCH_CACHE_DIR`     |                     | Path to a cache directory containing launch layers
| `<launcher>`        |                            | `/cnb/lifecycle/launcher` | Path to the `launcher` executable
| `<layers>`          | `CNB_LAYERS_DIR`           | `/layers`           | Path to layer directory
| `<log-level>`       | `CNB_LOG_LEVEL`            | `info`              | Log Level
| `<process-type>`    | `CNB_PROCESS_TYPE`         |                     | Default process type to set in the exported image
| `<project-metadata>`| `CNB_PROJECT_METADATA_PATH`| `<layers>/project-metadata.toml` | Path to a project metadata file (see [`project-metadata.toml`](#project-metadatatoml-toml)
| `<report>`          | `CNB_REPORT_PATH`          | `<layers>/report.toml`    | Path to report (see [`report.toml`](#reporttoml-toml)
| `<run-image>`       | `CNB_RUN_IMAGE`            | resolved from `<stack>`   | Run image reference
| `<stack>`           | `CNB_STACK_PATH`           | `/cnb/stack.toml`   | Path to stack file (see [`stack.toml`](#stacktoml-toml)
| `<uid>`             | `CNB_USER_ID`              |                     | UID of the stack `User`
| `<layers>/config/metadata.toml` | | | Build metadata (see [`metadata.toml`](#metadatatoml-toml)

- At least one `<image>` must be provided
- Each `<image>` MUST be a valid tag reference
- **If** `<daemon>` is `false` and more than one `<image>` is provided they MUST refer to the same registry
- **If** `<run-image>` is not provided by the platform the value will be [resolved](#run-image-resolution) from the contents of `stack`

##### Outputs
| Output             | Description
|--------------------|----------------------------------------------
| `[exit status]`    | Success (0), or error (1+)
| `/dev/stdout`      | Logs (info)
| `/dev/stderr`      | Logs (warnings, errors)
| `<image>`          | Exported app image (see [Buildpack Interface Specfication](buildpack.md)

| Exit Code | Result|
|-----------|-------|
| `0`       | Success
| `11`      | Platform API incompatibility error
| `12`      | Buildpack API incompatibility error
| `1-10`, `13-99` | Generic lifecycle errors
| `500-599`|  Export-specific lifecycle errors

- The lifecycle SHALL write the same app image to each `<image>` tag
- The app image:
    - MUST be an extension of the `<run-image>`
      - All run-image layers SHALL be preserved
      - All run-image config values SHALL be preserved unless this conflict with another requirement
    - MUST contain all buildpack-provided launch layers as determined by the [Buildpack Interface Specfication](buildpack.md)
    - MUST contain one or more app layers as determined by the [Buildpack Interface Specfication](buildpack.md)
    - MUST contain one or more launcher layers that include:
        - A file with the contents of the `<launcher>` file at path `/cnb/lifecycle/launcher`
        - One symlink per buildpack-provided process type with name `/cnb/process/<type>` and target `/cnb/lifecycle/launcher`
    - MUST contain a layer that includes `<layers>/config/metadata.toml`
    - **If** `<process-type>` matches a buildpack-provided process:
      - MUST have `ENTRYPOINT=/cnb/process/<process-type>`
    - **If** `<process-type>` does not match a buildpack-provided process:
      - **If** there is exactly one buildpack-provided process:
        - MUST have `ENTRYPOINT=/cnb/process/<type>` where `<type>` matches the `type` of the process
      - **Else**:
        - MUST have `ENTRYPOINT` set to `/cnb/lifecycle/launcher`
    - MUST contain the following `Env` entries
      - `CNB_LAYERS_DIR=<layers>`
      - `CNB_APP_DIR=<app>`
      - `PATH=/cnb/process:$PATH` where `$PATH` is the value of `$PATH` on the run-image.
    - MUST contain the following labels
        - `io.buildpacks.lifecycle.metadata`: see [lifecycle metadata label](#iobuildpackslifecyclemetadata-json)
        - `io.buildpacks.project.metadata`: the value of which SHALL be the json representation `<project-metadata>`
        - `io.buildpacks.build.metadata`: see [build metadata](#iobuildpacksbuildmetadata-json)
- To ensure [build reproducibility](#build-reproducibility), the lifecycle:
    - SHOULD set the modification time of all files in newly created layers to a constant value
    - SHOULD set the `created` time in image config to a constant value

- The lifecycle SHALL write a [report](#reporttoml-toml) to `<report>` describing the exported app image

- *If* a cache is provided the lifecycle:
   - SHALL write the contents of all cached layers to the cache
   - SHALL record the diffID and layer content metadata of all cached layers in the cache

#### `creator`
The platform MUST execute `creator` in the **build environment**

Usage:
```
/cnb/lifecycle/creator \
  [-app <app>] \
  [-buildpacks <buildpacks>] \
  [-cache-dir <cache-dir>] \
  [-cache-image <cache-image>] \
  [-daemon] \ # sets <daemon>
  [-gid <gid>] \
  [-launch-cache <launch-cache> ] \
  [-launcher <launcher> ] \
  [-layers <layers>] \
  [-log-level <log-level>] \
  [-order <order>] \
  [-platform <platform>] \
  [-previous-image <previous-image> ] \
  [-process-type <process-type> ] \
  [-project-metadata <project-metadata> ] \
  [-report <report> ] \
  [-run-image <run-image>] \
  [-skip-restore <skip-restore>] \
  [-stack <stack>] \
  [-tag <tag>...] \
  [-uid <uid> ] \
   <image>
```

##### Inputs
Running `creator` SHALL be equivalent to running `detector`, `analzyer`, `restorer`, `builder` and `exporter` in order with identical inputs where they are accepted, with the following exceptions.

| Input             | Environment Variable| Default Value| Description
|-------------------|---------------------|--------------|----------------------
| `<previous-image>`| `CNB_PREVIOUS_IMAGE`| `<image>`    | Image reference to be analyzed (usually the result of the previous build)
| `<skip-restore>`  | `CNB_SKIP_RESTORE`  | `false`      | Do not write layer metadata or restore cached layers
| `<tag>...`        |                     |              | Additional tag to apply to exported image

- **If** `<skip-restore>` is `true` the `creator` SHALL skip layer analysis and skip the entire Restore phase.
- **If** the platform provides one or more `<tag>` inputs they SHALL be treated as additional `<image>` inputs to the `exporter`

##### Outputs
Outputs produced by `creator` are identical to those produced by `exporter`, with the following additional expanded set of error codes.

| Exit Code | Result|
|-----------|-------|
| `0`       | Success
| `11`      | Platform API incompatibility error
| `12`      | Buildpack API incompatibility error
| `1-10`, `13-99` | Generic lifecycle errors
| `100-199`|  Detection-specific lifecycle errors
| `200-299`|  Analysis-specific lifecycle errors
| `300-399`|  Restoration-specific lifecycle errors
| `400-499`|  Build-specific lifecycle errors
| `500-599`|  Export-specific lifecycle errors


#### `rebaser`
Usage:
```
/cnb/lifecycle/rebaser \
  [-daemon] \ # sets <daemon>
  [-gid <gid>] \
  [-log-level <log-level>] \
  [-report <report> ] \
  [-run-image <run-image> | -image <run-image> ] \ # -image is Deprecated
  [-uid <uid>] \
  <image> [<image>...]
```

##### Inputs
| Input               | Environment Variable  | Default Value          | Description
|---------------------|-----------------------|------------------------|---------------------------------------
| `<daemon>`          | `CNB_USE_DAEMON`      | `false`                | Export image to docker daemon
| `<gid>`             | `CNB_GROUP_ID`        |                        | Primary GID of the stack `User`
| `<image>`           |                       |                        | App image to rebase
| `<log-level>`       | `CNB_LOG_LEVEL`       | `info`                 | Log Level
| `<report>`          | `CNB_REPORT_PATH`     | `<layers>/report.toml` | Path to report (see [`report.toml`](#reporttoml-toml)
| `<run-image>`       | `CNB_RUN_IMAGE`       | derived from `<image>` | Run image reference
| `<uid>`             | `CNB_USER_ID`         |                        | UID of the stack `User`

- At least one `<image>` must be provided
- Each `<image>` MUST be a valid tag reference
- **If** `<daemon>` is `false` and more than one `<image>` is provided they MUST refer to the same registry
- **If** `<run-image>` is not provided by the platform the value will be [resolved](#run-image-resolution) from the contents of the `stack` key in the `io.buildpacks.lifecycle.metdata` label on `<image>`.

##### Outputs
| Output             | Description
|--------------------|----------------------------------------------
| [exit status]      | (see Exit Code table below for values)
| `/dev/stdout`      | Logs (info)
| `/dev/stderr`      | Logs (warnings, errors)
| `<image>`          | Rebased app image (see [Buildpack Interface Specfication](buildpack.md)

| Exit Code | Result|
|-----------|-------|
| `0`       | Success
| `11`      | Platform API incompatibility error
| `12`      | Buildpack API incompatibility error
| `1-10`, `13-99` | Generic lifecycle errors
| `600-699`|  Rebase-specific lifecycle errors

- The lifecycle SHALL write the same app image to each `<image>` tag
- The rebased app image SHALL be identical to `<image>`, with the following modifications:
    - Run image layers SHALL be defined as Layers in `<image>` up to and including the layer with diff ID matching the value of `run-image.top-layer` from the `io.buildpacks.lifecycle.metadata` label
    - Run image layers SHALL be replaced with the layers from the new `<run-image>`
    - The value of `io.buildpacks.lifecycle.metadata` SHALL be modified as follows
      - `run-image.reference` SHALL uniquely identify `<run-image>`
      - `run-image.top-layer` SHALL be set to the uncompressed digest of the top layer in `<run-image>`
    - The value of `io.buildpacks.stack.*` labels SHALL be modified to that of the new `run-image`
- To ensure [build reproducibility](#build-reproducibility), the lifecycle:
    - Set the `created` time in image config to a constant

- The lifecycle SHALL write a [report](#reporttoml-toml) to `<report>` describing the rebased app image

#### `launcher`
Usage:
```
/cnb/process/<process-type> [<arg>...]
# OR
/cnb/lifecycle/launcher [--] [<cmd> <arg>...]
```
##### Inputs
| Input               | Environment Variable  | Default Value  | Description
|---------------------|-----------------------|----------------|---------------------------------------
| `<app>`             | `CNB_APP_DIR`         | `/workspace`   | Path to application directory
| `<layers>`          | `CNB_LAYERS_DIR`      | `/layers`      | Path to layer directory
| `<process-type>`    |                       |                | `type` of process to launch
| `<direct>`          |                       |                | Process execution strategy
| `<cmd>`             |                       |                | Command to execute
| `<args>`            |                       |                | Arguments to command
| `<layers>/config/metadata.toml`    |        |                | Build metadata (see [`metadata.toml`](#metadatatoml-toml)
| `<layers>/<buildpack-id>/<layer>/` |        |                | Launch Layers

A command (`<cmd>`), arguments to that command (`<args>`), and an execution strategy (`<direct>`) comprise a process definition. Processes MAY be buildpack-defined or user-defined.

The launcher:
- MUST derive the values of `<cmd>`, `<args>`, and `<direct>` as follows:
- **If** the final path element in `$0`, matches the type of any buildpack-provided process type
    - `<process-type>` SHALL be the final path element in `$0`
    - The lifecycle:
        - MUST select the process with type equal to `<process-type>` from `<layers>/config/metadata.toml`
        - MUST append any user-provided `<args>` to process arguments
- **Else**
    - **If** `$1` is `--`
        - `<direct>` SHALL be `true`
        - `<cmd>` SHALL be `$2`
        - `<args>` SHALL be `${@3:}`
    - **Else**
        - `<direct>` SHALL be `false`
        - `<cmd>` SHALL be `$1`
        - `<args>` SHALL be `${@2:}`

##### Outputs
If the launcher errors before executing the process it will have one of the following error codes:

| Exit Code | Result|
|-----------|-------|
| `11`      | Platform API incompatibility error
| `12`      | Buildpack API incompatibility error
| `700-799`|  Launch-specific lifecycle errors

Otherwise, the exit code shall be the exit code of the launched process.

The launcher:
- MUST construct the process execution environment as described in [Launch Environment](#launch-environment)
- MUST execute the selected process as specified in the [Buildpack Interface Specfication](buildpack.md)
- SHOULD replace the lifecycle with the process in memory without forking it.

### Run Image Resolution

Given stack metadata containing `run-image.image` and a set of `run-image.mirrors`. The `<run-image>` for a given `<image>` shall be resolved as follows:
- **If** any of `run-image.image` or `run-image.mirrors` has a registry matching that of `<image>`, this value will become the `<run-image>`
- **If** none of `run-image.image` or `run-image.mirrors` has a registry matching that of `<image>`, `<run-image.image>` will become the `<run-image>`

### Registry Authentication

The platform MAY set `CNB_REGISTRY_AUTH`  in the lifecycle execution environment, where value of `CNB_REGISTRY_AUTH` MUST be valid JSON object and MAY contain any number of `<regsitry>` to `<auth-header>` mappings.
If `CNB_REGISTRY_AUTH` is set and `<registry>` matches the registry of an image reference, the lifecycle SHOULD set the value of the `Authorization` HTTP header to `<auth-header>` when attempting to read or write the image located at the given reference.

If `CNB_REGISTRY_AUTH` is unset and a docker [config.json](https://docs.docker.com/engine/reference/commandline/cli/#configjson-properties) file is present, the lifecycle SHOULD use the contents of this file to authenticate with any matching registry.
The lifecycle SHOULD adhere to established docker conventions when checking for the existence of or interpreting the contents of a `config.json` file.

The lifecycle MAY provide other mechanisms by which a platform can supply registry credentials.

The lifecycle MUST attempt to authenticate anonymously if no matching credentials are found.

## Buildpacks

### Buildpacks Directory Layout

The buildpacks directory MUST contain unarchived buildpacks such that:

- Each top-level directory is a buildpack ID.
- Each second-level directory is a buildpack version.

## Security Considerations

The platform SHOULD run each phase of the lifecycle in an isolated container to prevent untrusted app and buildpack code from accessing storage credentials needed during the export and analysis phases.
A more thorough explanation is provided in the [Buildpack Interface Specification](buildpack.md).

## Additional Guidance

### Environment
#### Buildpack Environment
##### Stack-Provided Variables
The following variables SHOULD be set in the lifecycle execution environment and SHALL be directly inherited by the buildpack without modification:
| Env Variable    | Description
|-----------------|--------------------------------------
| `CNB_STACK_ID`  | Chosen stack ID
| `HOME`          | Current user's home directory
    
The following variables SHOULD be set in the lifecycle execution environment and MAY be modified by prior buildpacks before they are provided to a given buildpack:

| Env Variable      | Layer Path   | Contents
|-------------------|--------------|------------------
| `PATH`            | `/bin`       | binaries
| `LD_LIBRARY_PATH` | `/lib`       | shared libraries
| `LIBRARY_PATH`    | `/lib`       | static libraries
| `CPATH`           | `/include`   | header files
| `PKG_CONFIG_PATH` | `/pkgconfig` | pc files

The platform SHOULD NOT assume any other stack-provided environment variables are inherited by the buildpack.

##### User-Provided Variables
User-provided environment variables MUST be supplied by the platform as files in the `<platform>/env/` directory.
Each file SHALL define a single environment variable, where the file name defines the key and the file contents define the value.

User-provided environment variables MAY be modified by prior buildpacks before they are provided to a given buildpack.

The platform SHOULD NOT set user-provided environment variables directly in the lifecycle execution environment.

#### Launch Environment
User-provided modifications to the process execution environment SHOULD be set directly in the lifecycle execution environment.

The process SHALL inherit both stack-provided and user-provided variables from the lifecycle execution environment with the following exceptions:
* `CNB_APP_DIR`, `CNB_LAYERS_DIR` and `CNB_PROCESS_TYPE` SHALL NOT be set in the process execution environment.
* `/cnb/process` SHALL be removed from the beginning of `PATH`.
* The lifecycle SHALL apply buildpack-provided modifications to the environment as outlined in the [Buildpack Interface Specification](buildpack.md).

### Caching

If caching is enabled the platform is responsible for providing the lifecycle with access to the correct cache.
Whenever possible, the platform SHOULD provide the same cache to each rebuild of a given app image.
Cache locality and availability MAY vary between platforms.

### Build Reproducibility
When given identical inputs all build and rebase operations:
   - SHOULD produce app images with identical imageIDs
   - **If** exporting directly to a registry
       - SHOULD produce app images with identical manifest digests
   - MAY output other non-reproducible artifacts

To achieve reproducibility the lifecycle SHOULD set the following to a constant, rather than an accurate value:
- file modification times in generated layers
- image creation time

Because compressions algorithms and manifest whitespace affect the image digest, an app image exported to the docker daemon and subsequently pushed to a registry MAY have a different digest than an app image exported directly to a registry by the lifecycle, even when all other inputs are held constant.

If buildpacks do not generate layer contents or layer metadata reproducibly, builds MAY NOT be reproducibile even when identical source code and buildpacks are provided to the lifecycle.

All app image labels SHOULD contain only reproducible values.

For more information on build reproducibility see [https://reproducible-builds.org/](https://reproducible-builds.org/)

## Data Format

### Files

#### `analyzed.toml` (TOML)

```toml
[image]
  reference = "<image reference>"

[metadata]
# layer metadata
```

Where:
- `image.reference` MUST be either a digest reference to an image in a docker registry or the ID of an image in a docker daemon
- `metadata` MUST be the TOML representation of the layer [metadata label](#iobuildpackslifecyclemetadata-json)

#### `group.toml` (TOML)

```toml
[[group]]
id = "<buildpack ID>"
version = "<buildpack version>"
api = "<buildpack API version>"
homepage = "<buildpack homepage>"
```

Where:

- `id`, `version`, and `api` MUST be present for each buildpack object in a group.

#### `metadata.toml` (TOML)
```toml
[[buildpacks]]
id = "<buildpack ID>"
version = "<buildpack version>"
api = "<buildpack API version>"
optional = false

[[processes]]
type = "<process type>"
command = "<command>"
args = ["<arguments>"]
direct = false

[[slices]]
paths = ["<app sub-path glob>"]

[bom]
```

Where:
- `id`, `version`, and `api` MUST be present for each buildpack.
- `processes` contains the complete set of processes contributed by all buildpacks
- `processes` contains the complete set of slice defined by all buildpacks
- `bom` contains the Bill of Materials

#### `order.toml` (TOML)

```toml
[[order]]
[[order.group]]
id = "<buildpack ID>"
version = "<buildpack version>"
optional = false
```

Where:

- Both `id` and `version` MUST be present for each buildpack object in a group.
- The value of `optional` MUST default to false if not specified.

#### `plan.toml` (TOML)
```toml
[[entries]]

  [[entries.providers]]
    id = "<buildpack ID>"
    version = "buildpack Version"

  [[entries.requires]]
    name = "<dependency name>"
    [entries.requires.metadata]
      # arbitrary data describing the required dependency
```
Where:
- `entries` MAY be empty
- Each entry:
    - MUST contain at least one buildpack in `providers`
    - MUST contain at least one dependency requirement in `requires`
    - MUST exclusively contain dependency requirements with the same `<dependency name>`

#### `project-metadata.toml` (TOML)

```toml
[source]
type = "<source type>"

[source.version]
# arbitrary data

[source.metadata]
# arbitrary data
```

Where:
- All values are optional
- `type`, if present, SHOULD contain the type of location where the provided app source is stored (e.g `git`, `s3`)
- `version`, if present, SHOULD contain data uniquely identifying the particular version of the provided source
- `metadata` MAY contain additional arbitrary data about the provided source

#### `report.toml` (TOML)
```toml
[image]
tags = ["<tag reference>"]
digest = "<image digest>"
image-id = "<imageID>"

[build]
[[build.bom]]
name = "<dependency name>"

[build.bom.metadata]
version = "<dependency version>"
```
Where:
- `tags` MUST contain all tag references to the exported app image
- **If** the app image was exported to an OCI registry
  - `digest` MUST contain the image digest
- **If** the app image was exported to a docker daemon
  - `imageID` MUST contain the imageID
- **If** the app image was the result of a build operation
  - `build.bom` MUST contain any build Bill-of-Materials entries returned by participating buildpacks

#### `stack.toml` (TOML)

```toml
[run-image]
 image = "<image>"
 mirrors = ["<mirror>", "<mirror>"]
```

Where:
- `run-image.image` MAY be a reference to a run image in a docker registry
- `run-image.mirrors` MUST NOT be present if `run-image.image` is not present
- `run-image.mirrors` MAY contain one or more tag references to run images in docker registries
- All `run-image.mirrors`:
  - SHOULD reference an image with ID identical to that of `run-image.image`
- `run-image.image` and `run-image.mirrors.[]` SHOULD each refer to a unique registry

### Labels

#### `io.buildpacks.build.metadata` (JSON)

```javascript
{
  "processes": [
    {
      "type": "<process-type>",
      "command": "<command>",
      "args": [
        "<args>"
      ],
      "direct": false
    }
  ],
  "buildpacks": [
    {
      "id": "<buildpack ID>",
      "version": "<buildpack version>",
      "homepage": "<buildpack homepage>"
    }
  ],
  "bom": [
    {
      "name": "<bom-entry-name>",
      "metadata": {
        // arbitrary buildpack provided metadata
      },
      "buildpack": {
        "id": "<buildpack ID>",
        "version": "<buildpack version>"
      }
    },
  ],
 "launcher": {
    "version": "<launcher-version>",
    "source": {
      "git": {
        "repository": "<launcher-source-repository>",
        "commit": "<launcher-source-commit>"
      }
    }
  }
}
```
Where:
- `processes` MUST contain all buildpack contributed processes
- `buildpacks` MUST contain the detected group
- `bom` MUST contain the Bill of Materials
- `launcher.version` SHOULD contain the version of the `launcher` binary included in the app
- `luancher.source.git.repository` SHOULD contain the git repository containing the `launcher` source code
- `luancher.source.git.commit` SHOULD contain the git commit from which the given `launcher` was built

#### `io.buildpacks.lifecycle.metadata` (JSON)

```javascript
{
  "app": [
    {"sha": "<slice-layer-diffID>"}
  ],
  "config": {
    "sha": "<config-layer-diffID>"
  },
  "launcher": {
    "sha": "<launcher-layer-diffID>"
  },
  "buildpacks": [
    {
      "key": "<buldpack-id>",
      "version": "<buildpack-version>",
      "layers": {
        "<layer-name>": {
          "sha": "<layer-diffID>",
          "data": {},
          "build": false,
          "launch": false,
          "cache": false
        }
      }
    }
  ],
  "runImage": {
    "topLayer": "<run-image-top-layer-diffID>",
    "reference": "<run-image-reference>"
  },
  "stack": {
    "runImage": {
      "image": "cnbs/sample-stack-run:bionic"
    }
  }
}
```
Where:
- `app` MUST contain one entry per app slice layer where
  - `sha` MUST contain the digest of the uncompressed layer
- `config.sha` MUST the digest of the uncompressed layer containing launcher config
- `launcher.sha` MUST the digest of the uncompressed layer containing the launcher binary
- `buildpacks` MUST contain one entry per buildpack that participated in the build where
  - `key` is required and MUST contain the buildpack ID
  - `version` is required and MUST contain the buidpack Version
  - `layers` is required and MUST contain one entry per launch layer contributed by the given buildpack.
  - For each entry in `layers`:
    - The key  MUST be the name of the layer
    - The value MUST contain JSON representation of the `layer.toml` with an additional `sha` key, containing the digest of the uncompressed layer
    - The value MUST contain an additional `sha` key, containing the digest of the uncompressed layer
- `run-image.topLayer` must contain the uncompressed digest of the top layer of the run-image
- `run-image.reference` MUST uniquely identify the run image. It MAY contain one of the following
  - An image ID (the digest of the uncompressed config blob)
  - A digest reference to a manifest stored in an OCI image registry
- `stack` MUST contain the json representation of `stack.toml`

#### `io.buildpacks.project.metadata` (JSON)

```javascript
{
  "source": {
    "type": "<type>",
    "version": {
     // arbitrary version data
    },
    "metadata": {
    // arbitrary data
    }
  }
}
```
This label MUST contain the JSON representation of [`project-metadata.toml`](#project-metadatatoml-toml)
