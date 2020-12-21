# Buildpack Interface Specification

This document specifies the interface between a lifecycle program and one or more buildpacks.

The lifecycle program uses buildpacks to build software artifacts from source code and pack the result into an OCI image.

This is accomplished in four required phases, and one optional phase:

1. **Detection,** where an optimal selection of compatible buildpacks is chosen.
1. **Analysis,** where metadata about OCI layers generated during a previous build are made available to buildpacks.
1. **Build,** where buildpacks use that metadata to generate only the OCI layers that need to be replaced.
1. **Extend,** (optional) where privileged buildpacks generate OCI layers that will be made available at runtime.
1. **Export,** where the remote layers are replaced by the generated layers.

The `ENTRYPOINT` of the OCI image contains logic implemented by the lifecycle that executes during the **Launch** phase.

## Table of Contents

<!-- Using https://github.com/yzhang-gh/vscode-markdown to manage toc -->
- [Buildpack Interface Specification](#buildpack-interface-specification)
  - [Table of Contents](#table-of-contents)
  - [Buildpack API Version](#buildpack-api-version)
  - [Types of Buildpacks](#types-of-buildpacks)
  - [Buildpack Interface](#buildpack-interface)
    - [Buildpack API Compatibility](#buildpack-api-compatibility)
    - [Key](#key)
    - [Detection](#detection)
    - [Build](#build)
    - [Exec.d](#execd)
    - [Application Layer Types](#application-layer-types)
      - [Launch Layers](#launch-layers)
      - [Build Layers](#build-layers)
      - [Cached Layers](#cached-layers)
      - [Other Layers](#other-layers)
    - [Stack Layers](#stack-layers)
  - [App Interface](#app-interface)
  - [Phase #1: Detection](#phase-1-detection)
    - [Purpose](#purpose)
    - [Process](#process)
      - [Order Resolution](#order-resolution)
      - [Mixin Resolution](#mixin-resolution)
  - [Phase #2: Analysis](#phase-2-analysis)
    - [Purpose](#purpose-1)
    - [Process](#process-1)
  - [Phase #3: Build](#phase-3-build)
    - [Purpose](#purpose-2)
    - [Process](#process-2)
      - [Unmet Buildpack Plan Entries](#unmet-buildpack-plan-entries)
      - [Bills-of-Materials](#bills-of-materials)
      - [Application Layers](#application-layers)
  - [Phase #4: Extend](#phase-4-extend)
    - [Purpose](#purpose-3)
    - [Process](#process-3)
  - [Phase #5: Export](#phase-5-export)
    - [Purpose](#purpose-4)
    - [Process](#process-4)
  - [Launch](#launch)
    - [Purpose](#purpose-5)
    - [Process](#process-5)
  - [Environment](#environment)
    - [Provided by the Lifecycle](#provided-by-the-lifecycle)
      - [Buildpack Specific Variables](#buildpack-specific-variables)
      - [Layer Paths](#layer-paths)
    - [Provided by the Platform](#provided-by-the-platform)
    - [Provided by the Buildpacks](#provided-by-the-buildpacks)
      - [Environment Variable Modification Rules](#environment-variable-modification-rules)
        - [Append](#append)
        - [Default](#default)
        - [Delimiter](#delimiter)
        - [Override](#override)
        - [Prepend](#prepend)
  - [Security Considerations](#security-considerations)
    - [Assumptions of Trust](#assumptions-of-trust)
    - [Requirements](#requirements)
  - [Data Format](#data-format)
    - [launch.toml (TOML)](#launchtoml-toml)
    - [build.toml (TOML)](#buildtoml-toml)
    - [store.toml (TOML)](#storetoml-toml)
    - [Build Plan (TOML)](#build-plan-toml)
    - [Buildpack Plan (TOML)](#buildpack-plan-toml)
    - [Layer Content Metadata (TOML)](#layer-content-metadata-toml)
    - [buildpack.toml (TOML)](#buildpacktoml-toml)
      - [Buildpack Implementations](#buildpack-implementations)
      - [Order Buildpacks](#order-buildpacks)
    - [Exec.d Output (TOML)](#execd-output-toml)
  - [Deprecations](#deprecations)
    - [`0.3`](#03)
      - [Build Plan (TOML) `requires.version` Key](#build-plan-toml-requiresversion-key)

## Buildpack API Version
This document specifies Buildpack API version `0.5`

Buildpack API versions:
 - MUST be in form `<major>.<minor>` or `<major>`, where `<major>` is equivalent to `<major>.0`
 - When `<major>` is greater than `0` increments to `<minor>` SHALL exclusively indicate additive changes

## Types of Buildpacks

- `application buildpack` - a traditional buildpack that does not run as root, and has access to the app being built.
- `stack buildpack` - a type of buildpack that runs with root privileges against the stack image(s) instead of an app. Stack buildpacks must not make changes to the build and run images that either violate stack compatibility guarantees or violate the contract defined by that stack's author.

## Buildpack Interface

The following specifies the interface implemented by executables in each buildpack.
The lifecycle MUST invoke these executables as described in the Phase sections.

### Buildpack API Compatibility
Given a buildpack declaring `<buildpack API Version>` in its [`buildpack.toml`](#buildpacktoml-toml), the lifecycle:
- MUST either conform to the matching version of this specification when interfacing with the buildpack or
- return an error to the platform if it does not support `<buildpack API Version>`

The lifecycle MAY return an error to the platform if two or more buildpacks within a group declare buildpack API versions that the lifecycle cannot support together within a single build, even if both are supported independently.

### Key

| Mark | Meaning
|------|-------------------------------------------
| A    | Single copy provided for all buildpacks
| E    | Different copy provided for each buildpack
| I    | Image repository for storage
| C    | Cache for storage
| R    | Read-only
| *    | Buildpack-specific content
| #    | Platform-specific content

### Detection

Executable: `/bin/detect <platform[AR]> <plan[E]>`, Working Dir: `<app[AR]>`

| Input             | Description
|-------------------|----------------------------------------------
| `$0`              | Absolute path of `/bin/detect` executable
| `<platform>/env/` | User-provided environment variables for build
| `<platform>/#`    | Platform-specific extensions

| Output             | Description
|--------------------|----------------------------------------------
| [exit status]      | Pass (0), fail (100), or error (1-99, 101+)
| Standard output    | Logs (info)
| Standard error     | Logs (warnings, errors)
| `<plan>`           | Contributions to the the Build Plan (TOML)


###  Build

Executable: `/bin/build <layers[EIC]> <platform[AR]> <plan[ER]>`, Working Dir: `<app[AI]>`

| Input             | Description
|-------------------|----------------------------------------------
| `$0`              | Absolute path of `/bin/build` executable
| `<plan>`          | Relevant [Buildpack Plan entries](#buildpack-plan-toml) from detection (TOML)
| `<platform>/env/` | User-provided environment variables for build
| `<platform>/#`    | Platform-specific extensions

#### Application Buildpack Outputs

| Output                                   | Description
|------------------------------------------|--------------------------------------
| [exit status]                            | Success (0) or failure (1+)
| Standard output                          | Logs (info)
| Standard error                           | Logs (warnings, errors)
| `<layers>/launch.toml`                   | App metadata (see [launch.toml](#launchtoml-toml))
| `<layers>/build.toml`                    | Build metadata (see [build.toml](#buildtoml-toml))
| `<layers>/store.toml`                    | Persistent metadata (see [store.toml](#storetoml-toml))
| `<layers>/<layer>.toml`                  | Application layer metadata (see [Layer Content Metadata](#layer-content-metadata-toml))
| `<layers>/<layer>/bin/`                  | Binaries for launch and/or subsequent buildpacks
| `<layers>/<layer>/lib/`                  | Shared libraries for launch and/or subsequent buildpacks
| `<layers>/<layer>/profile.d/`            | Scripts sourced by Bash before launch
| `<layers>/<layer>/profile.d/<process>/`  | Scripts sourced by Bash before launch for a particular process type
| `<layers>/<layer>/exec.d/`               | Executables that provide env vars via the [Exec.d Interface](#execd) before launch
| `<layers>/<layer>/exec.d/<process>/`     | Executables that provide env vars for a particular process type via the [Exec.d Interface](#execd) before launch
| `<layers>/<layer>/include/`              | C/C++ headers for subsequent buildpacks
| `<layers>/<layer>/pkgconfig/`            | Search path for pkg-config for subsequent buildpacks
| `<layers>/<layer>/env/`                  | Env vars for launch and/or subsequent buildpacks
| `<layers>/<layer>/env.launch/`           | Env vars for launch (after `env`, before `profile.d`)
| `<layers>/<layer>/env.launch/<process>/` | Env vars for launch (after `env`, before `profile.d`) for the launched process
| `<layers>/<layer>/env.build/`            | Env vars for subsequent buildpacks (after `env`)
| `<layers>/<layer>/*`                     | Other content for launch and/or subsequent buildpacks

#### Stack Buildpack Outputs

| Output                                   | Description
|------------------------------------------|--------------------------------------
| [exit status]                            | Success (0) or failure (1+)
| Standard output                          | Logs (info)
| Standard error                           | Logs (warnings, errors)
| `<layers>/stack-layer.toml`              | Stack layer metadata (see [stack-layer.toml](#stack-layer-content-metadata-toml))

### Exec.d

Executable: `<layers>/<layer>/exec.d/<executable>`, Working Dir: `<app[AI]>`

OR

Executable: `<layers>/<layer>/exec.d/<process>/<executable>`, Working Dir: `<app[AI]>`

| Input             | Description
|-------------------|----------------------------------------------
| `$0`              | Absolute path of the executable
| FD 3              | A third open file [†](README.md#linux-only)descriptor or [‡](README.md#windows-only)handle

| Output             | Description
|--------------------|----------------------------------------------
| [exit status]      | Pass (0) or error (1+)
| Standard output    | Logs (info)
| Standard error     | Logs (warnings, errors)
| FD 3               | Launch time environment variables (see [Exec.d Output](#execd-output-toml))

### Application Layer Types

Application layers are defined as layers created from a `<layers>/<layer>.toml` file. The lifecycle MUST accept either [Application Layer Content Metadata](#layer-content-metadata-toml) defined in a set of `<layers>/<layer>.toml` files OR a [Stack Layer](#stack-layer) defined by a `<layers>/stack-layer.toml` as output from a buildpack, but not both.

Using the [Layer Content Metadata](#layer-content-metadata-toml) provided by a buildpack in a `<layers>/<layer>.toml` file, the lifecycle MUST determine:

- Whether the layer directory in `<layers>/<layer>/` should be available to the app (via the `launch` boolean).
- Whether the layer directory in `<layers>/<layer>/` should be available to subsequent buildpacks (via the `build` boolean).
- Whether and how the layer directory in `<layers>/<layer>/` should be persisted to subsequent builds of the same OCI image (via the `cache` boolean).

All combinations of `launch`, `build`, and `cache` booleans are valid. When a layer declares more than one type (e.g. `launch = true` and `cache = true`), the requirements of each type apply.

#### Launch Layers

A buildpack MAY specify that a `<layers>/<layer>/` directory is a launch layer by placing `launch = true` in `<layers>/<layer>.toml`.

The lifecycle MUST make all launch layers accessible to the app as described in the [Environment](#environment) section.

The lifecycle MUST include each launch layer in the built OCI image.
The lifecycle MUST also store the Layer Content Metadata associated with each layer so that it can be recovered using the layer Diff ID.

Before a given re-build:
- If a launch layer is marked `cache = false` and `build = false`, the lifecycle:
  - MUST restore the entire `<layers>/<layer>.toml` file from the previous build to the same path and
  - MUST NOT restore the corresponding `<layers>/<layer>/` directory from any previous build.

After a given re-build:
- If a buildpack keeps `launch = true` in `<layers>/<layer>.toml` and leaves no `<layers>/<layer>/` directory, the lifecycle:
  - MUST reuse the corresponding layer from the previous build in the OCI image and
  - MUST replace the Layer Content Metadata in the OCI image with the version present after the re-build.
- If a buildpack keeps `launch = true` in `<layers>/<layer>.toml` and leaves a `<layers>/<layer>/` directory, the lifecycle:
  - MUST replace the corresponding layer in the OCI image with the directory contents present after the re-build and
  - MUST replace the Layer Content Metadata in the OCI image with the version present after the re-build.
- If a buildpack removes `launch = true` from `<layers>/<layer>.toml` or deletes `<layers>/<layer>.toml`, then the lifecycle MUST NOT include any corresponding layer in the OCI image.

#### Build Layers

A buildpack MAY specify that a `<layers>/<layer>/` directory is a build layer by placing `build = true` in `<layers>/<layer>.toml`.

The lifecycle MUST make all build layers accessible to subsequent buildpacks as described in the [Environment](#environment) section.

Before the next re-build:
- If the layer is marked `cache = false`, the lifecycle MUST NOT restore the `<layers>/<layer>/` directory or the `<layers>/<layer>.toml` file from any previous build.

#### Cached Layers

A buildpack MAY specify that a `<layers>/<layer>/` directory is a cached layer by placing `cache = true` in `<layers>/<layer>.toml`.

If a cache is provided the lifecycle:
- SHOULD store all cached layers after a successful build.
- SHOULD store the Layer Content Metadata associated with each layer so that it can be recovered using the layer Diff ID

Before the next re-build:
- The lifecycle:
  - MUST either restore the entire `<layers>/<layer>.toml` file and corresponding `<layers>/<layer>/` directory from any previous build to the same paths or
  - MUST restore neither the `<layers>/<layer>.toml` file nor corresponding `<layers>/<layer>/` directory.

#### Other Layers

The lifecycle:
- If `build = false`
  - MUST NOT make the layer available to subesequent buildpacks.
- If `launch = false`
  - MUST NOT include the layer or the Layer Content Metadata in the OCI image.
- If `cache = false`
  - MUST NOT store the layer or the Layer Content Metadata in the cache.

Layers marked `launch = false`, `build = false`, and `cache = false` behave like temporary directories, available only to the authoring buildpack, that exist for the duration of a single build.

### Stack Layers

All changes made by a stack buildpack in the extend phase MAY be included in a single layer produced as output from the buildpack, which MUST be mounted into the `/run-layers` directory of the export container. Any changes performed by the stack buildpack to the build image MUST persist through execution of application buildpacks, but MUST NOT be exported as a layer.

The stack buildpack's layer MAY be enriched by providing [Stack Layer Content Metadata](#stack-layer-content-metadata-toml) in a `<layers>/stack-layer.toml` file. The `<layers>/stack-layer.toml` MAY define globs of files to be excluded from the image when it is _exported_. Any excluded path MAY also be marked as _cached_, so that those excluded paths are recovered before the build or extend phase. The term _excluded_ is defined as:

* *Excluded during build phase*: Files at matching paths are removed from the filesystem before application buildpack execution. If cached, on subsequent build operations, the excluded files will be restored to the build-image filesystem before stackpack execution.
* *Excluded during extend phase*: A given path is excluded from the snapshot layer (and thus from the final image). If cached, on subsequent build or rebase operations, the excluded files will be restored to the run-image filesystem before stackpack execution.

## App Interface

| Output                 | Description
|------------------------|----------------------------------------------
| `<app>/.profile`       | [†](README.md#linux-only) Bash-formatted script sourced by shell before launch
| `<app>/.profile.bat`   | [‡](README.md#windows-only) BAT-formatted script sourced by shell before launch

## Phase #1: Detection

![Detection](img/detection.svg)

### Purpose

The purpose of detection is to find an ordered group of buildpacks to use during the build phase.
These buildpacks must be compatible with the app.

### Process

**GIVEN:**
- A single group of stack buildpack resolved into buildpack implementations as described in [Order Resolution](#order-resolution)
- An ordered list of application buildpack groups resolved into buildpack implementations as described in [Order Resolution](#order-resolution)
- A directory containing application source code
- A shell, if needed,

For each group in the ordered list of application buildpack groups, the stack buildpack group MAY be appended to the begining of the application buildpack group with the stack buildpacks as _optional_ buildpacks in the resulting group.

For each buildpack in each group in order, the lifecycle MUST execute `/bin/detect`.

1. **If** the exit status of `/bin/detect` is non-zero and the buildpack is not marked optional, \
   **Then** the lifecycle MUST proceed to the next group or fail detection completely if no more groups are present.

2. **If** the exit status of `/bin/detect` is zero or the buildpack is marked optional,
   1. **If** the buildpack is not the last buildpack in the group, \
      **Then** the lifecycle MUST proceed to the next buildpack in the group.

   2. **If** the buildpack is the last buildpack in the group,
      1. **If** no exit statuses from `/bin/detect` in the group are zero \
         **Then** the lifecycle MUST proceed to the next group or fail detection completely if no more groups are present.

      2. **If** at least one exit status from `/bin/detect` in the group is zero \
         **Then** the lifecycle MUST select this group and proceed to the analysis phase.

The selected group MUST be filtered to only include buildpacks with exit status zero.
The order of the buildpacks in the group MUST otherwise be preserved.

The `/bin/detect` executable in each buildpack, when executed:

- MAY read the app directory.
- MAY read the detect environment as described in the [Environment](#environment) section.
- MAY emit error, warning, or debug messages to `stderr`.
- MAY augment the Build Plan by writing TOML to `<plan>`.
- MUST set an exit status code as described in the [Buildpack Interface](#buildpack-interface) section.

In order to make contributions to the Build Plan, a `/bin/detect` executable MUST write entries to `<plan>` in two sections: `requires` and `provides`.
Additionally, these two sections MAY be repeated together inside of an `or` array at the top-level.
Each `requires` and `provides` section MUST be a list of entries formatted as described in the [Build Plan](#build-plan-toml) format section.

Each pairing of `requires` and `provides` sections (at the top level, or inside of an `or` array) is a potential Build Plan.

A stack buildpack MAY NOT require any entries in the build plan.

For a given buildpack group, a sequence of trials is generated by selecting a single potential Build Plan from each buildpack in a left-to-right, depth-first order.
The group fails to detect if all trials fail to detect.

Each dependency in the `requires` and `provides` MAY be a mixin. A dependency that does not specify a mixin is called a _non-mixin dependency_. A application buildpack MAY NOT provide mixins in the build plan. A application buildpack MAY require mixins in the build plan.

For each trial,
- If a required buildpack provides a non-mixin dependency that is not required by the same buildpack or a subsequent buildpack, the trial MUST fail to detect.
- If a required buildpack requires a non-mixin dependency that is not provided by the same buildpack or a previous buildpack, the trial MUST fail to detect.
- If an optional buildpack provides a non-mixin dependency that is not required by the same buildpack or a subsequent buildpack, the optional buildpack MUST be excluded from the build phase and its requires and provides MUST be excluded from the Build Plan.
- If an optional buildpack requires a non-mixin dependency that is not provided by the same buildpack or a previous buildpack, the optional buildpack MUST be excluded from the build phase and its requires and provides MUST be excluded from the Build Plan.
- Multiple buildpacks MAY require or provide the same dependency.
- Any mixin dependencies MUST be resolved as described in [Mixin Resolution](#mixin-resolution), and the trail shall pass or fail detect accordingly.

The lifecycle MAY execute each `/bin/detect` within a group in parallel.

The lifecycle MUST run `/bin/detect` for all buildpacks in a group in a container using common stack with a common set of mixins.
The lifecycle MUST fail detection if any of those buildpacks does not list that stack in `buildpack.toml`.
The lifecycle MUST fail detection if any of those buildpacks specifies a mixin associated with that stack in `buildpack.toml` that is unavailable in the container.

#### Order Resolution

During detection, an order definition MUST be resolved into individual buildpack implementations.

The resolution process MUST follow this pattern:

Where:
- O and P are buildpack orders.
- A through H are buildpack implementations.

Given:

<img src="http://tex.s2cms.ru/svg/%0AO%20%3D%0A%5Cbegin%7Bbmatrix%7D%0AA%2C%20%26%20B%20%5C%5C%0AC%2C%20%26%20D%0A%5Cend%7Bbmatrix%7D%0A" alt="
O =
\begin{bmatrix}
A, &amp; B \\
C, &amp; D
\end{bmatrix}
" />

<img src="http://tex.s2cms.ru/svg/%0AP%20%3D%0A%5Cbegin%7Bbmatrix%7D%0AE%2C%20%26%20F%20%5C%5C%0AG%2C%20%26%20H%0A%5Cend%7Bbmatrix%7D%0A" alt="
P =
\begin{bmatrix}
E, &amp; F \\
G, &amp; H
\end{bmatrix}
" />

We propose:

<img src="http://tex.s2cms.ru/svg/%0A%5Cbegin%7Bbmatrix%7D%0AE%2C%20%26%20O%2C%20%26%20F%0A%5Cend%7Bbmatrix%7D%20%3D%20%0A%5Cbegin%7Bbmatrix%7D%0AE%2C%20%26%20A%2C%20%26%20B%2C%20%26%20F%20%5C%5C%0AE%2C%20%26%20C%2C%20%26%20D%2C%20%26%20F%20%5C%5C%0A%5Cend%7Bbmatrix%7D%0A" alt="
\begin{bmatrix}
E, &amp; O, &amp; F
\end{bmatrix} =
\begin{bmatrix}
E, &amp; A, &amp; B, &amp; F \\
E, &amp; C, &amp; D, &amp; F \\
\end{bmatrix}
" />

<img src="http://tex.s2cms.ru/svg/%0A%5Cbegin%7Bbmatrix%7D%0AO%2C%20%26%20P%0A%5Cend%7Bbmatrix%7D%20%3D%20%0A%5Cbegin%7Bbmatrix%7D%0AA%2C%20%26%20B%2C%20%26%20E%2C%20%26%20F%20%5C%5C%0AA%2C%20%26%20B%2C%20%26%20G%2C%20%26%20H%20%5C%5C%0AC%2C%20%26%20D%2C%20%26%20E%2C%20%26%20F%20%5C%5C%0AC%2C%20%26%20D%2C%20%26%20G%2C%20%26%20H%20%5C%5C%0A%5Cend%7Bbmatrix%7D%0A" alt="
\begin{bmatrix}
O, &amp; P
\end{bmatrix} =
\begin{bmatrix}
A, &amp; B, &amp; E, &amp; F \\
A, &amp; B, &amp; G, &amp; H \\
C, &amp; D, &amp; E, &amp; F \\
C, &amp; D, &amp; G, &amp; H \\
\end{bmatrix}
" />

Note that buildpack IDs are expanded depth-first in left-to-right order.

If a buildpack order entry within a group has the parameter `optional = true`, then a copy of the group without the entry MUST be repeated after the original group.

#### Mixin Resolution

During detection, a list of required mixins MUST be resolved against a list of provided mixins.

Mixins MAY ONLY BE required from the following sources:
- The [Build Plan](#build-plan-toml) of a buildpack
- The [Buildpack Descriptor](#buildpacktoml-toml) of a buildpack

Mixins MAY ONLY BE provided by the following sources:
- The stack (run-image and/or build-image)
- The [Buildpack Descriptor](#buildpacktoml-toml) of a buildpack

If any required mixins are not also provided, then the group MUST fail to detect.

If a stack provides mixins that are not required by any buildpacks, the group MAY pass detection.

If a stack buildpack provides a mixin that is not required, the stack buildpack MAY pass detection. For each of the build and run phases:
* If a stack buildpack provides mixins, and at least one of those mixins are required; it MAY pass
* If a stack buildpack provides mixins, and none of those mixins are required; it MUST be skipped
* If a stack buildpack provides mixins, and none of those mixins are required, but it also provides another dependency (non-mixin), which is required; it MAY pass following the normal build plan rules
* If a stack buildpack does not provide mixins; it MAY pass

If a mixin is required for a single stage only with the `build:` or `run:` prefix, it MUST NOT be included in the Buildpack Build Plan during the stage where it is not required. During the detect phase, the lifecycle MUST create a build plan containing only the entries required during that stage (build or run) without the stage-specifier prefix.
* If a mixin is required for "run" stage only, it MUST NOT appear in the buildpack plan entries during build
* If a mixin is required for "build" stage only, it MUST NOT appear in the buildpack plan entries during extend

## Phase #2: Analysis

![Analysis](img/analysis.svg)

### Purpose

The purpose of analysis is to restore `<layers>/<layer>.toml` and `<layers>/store.toml` files that buildpacks may use to optimize the build and export phases.

### Process

The lifecycle SHOULD attempt to locate a reference to an OCI image from a previous build that:

- Was created using some version of the same application source code.
- Is readable by the lifecycle.
- Was created using the lifecycle.
- Is as recent as possible.

The lifecycle MUST skip analysis and proceed to the build phase if no such image can be located.

**GIVEN:**
- A reference to the previously created OCI image described above and
- The final ordered group of buildpacks determined during the detection phase,

For each buildpack in the group,

1. All `<layers>/<layer>.toml` files with `cache = true` and corresponding `<layers>/<layer>` directories from any previous build are restored to their same filesystem locations.
2. Each `<layers>/<layer>.toml` file with `launch = true` and `cache = false` that was present at the end of the build of the previously created OCI image is retrieved.
3. A given `<layers>/<layer>.toml` file with `launch = true` and `cache = true` and corresponding  `<layers>/<layer>` directory are both removed if either do not match their contents in the previously created OCI image.

Finally, the contents of `<layers>/store.toml` from the build of the previously created OCI image are restored.

After analysis, the lifecycle MUST proceed to the build phase.

## Phase #3: Build

![Build](img/build.svg)

### Purpose

The purpose of build is to transform application source code into runnable artifacts that can be packaged into a container.

During the build phase, typical buildpacks might:

1. Read the Buildpack Plan in `<plan>` to determine what dependencies to provide.
2. Provide the application with dependencies for launch in `<layers>/<layer>`.
3. Provide subsequent buildpacks with dependencies in `<layers>/<layer>`.
4. Compile the application source code into object code.
5. Remove application source code that is not necessary for launch.
6. Provide start command in `<layers>/launch.toml`.
7. Write a partial Bill-of-Material to `<layers>/launch.toml` describing any provided application dependencies.
8. Write a partial Bill-of-Material to `<layers>/build.toml` describing any provided build dependencies.

The purpose of separate `<layers>/<layer>` directories is to:

- Minimize the execution time of the build.
- Minimize usage of network communications.
- Minimize persistent disk usage.

This is achieved by:

- Reducing the number of necessary build operations during the build phase.
- Reducing data transfer during the export phase.
- Enabling de-duplication of stored image layers.

### Process

**GIVEN:**
- The final ordered group of buildpacks determined during the detection phase,
- A directory containing application source code,
- The Buildpack Plan,
- Any `<layers>/<layer>.toml` files placed on the filesystem during the analysis phase,
- Any locally cached `<layers>/<layer>` directories, and
- A shell, if needed,

For each buildpack in the group in order, the lifecycle MUST execute `/bin/build`.

1. **If** the exit status of `/bin/build` is non-zero, \
   **Then** the lifecycle MUST fail the build.

2. **If** the exit status of `/bin/build` is zero,
   1. **If** there are additional buildpacks in the group, \
      **Then** the lifecycle MUST proceed to the next buildpack's `/bin/build`.

   2. **If** there are no additional buildpacks in the group, \
      **Then** the lifecycle MUST proceed to the next phase.

For each `/bin/build` executable in each buildpack, the lifecycle:

- MUST provide path arguments to `/bin/build` as described in the [Buildpack Interface](#buildpack-interface) section.
- MUST configure the build environment as described in the [Environment](#environment) section.
- MUST provide all `<plan>` entries that were required by any buildpack in the group during the detection phase with names matching the names that the buildpack provided.

Correspondingly, each `/bin/build` executable:

- MAY read the build environment as described in the [Environment](#environment) section.
- MAY read the Buildpack Plan.
- SHOULD write a list containing any [Unmet Buildpack Plan Entries](#unmet-buildpack-plan-entries) to `<layers>/build.toml` to defer those entries to subsequent `/bin/build` executables.
- MAY log output from the build process to `stdout`.
- MAY emit error, warning, or debug messages to `stderr`.
- SHOULD write build BOM entries to `<layers>/build.toml` describing any contributions to the build environment.
- MAY write values that should persist to subsequent builds in `<layers>/store.toml`.

Each `/bin/build` executable of a application buildpack:

- MAY read or write to the `<app>` directory.
- MAY write a list of possible commands for launch to `<layers>/launch.toml`.
- MAY write a list of sub-paths within `<app>` to `<layers>/launch.toml`.
- SHOULD write BOM (Bill-of-Materials) entries to `<layers>/launch.toml` describing any contributions to the app image.
- MAY modify or delete any existing `<layers>/<layer>` directories.
- MAY modify or delete any existing `<layers>/<layer>.toml` files.
- MAY create new `<layers>/<layer>` directories.
- MAY create new `<layers>/<layer>.toml` files.
- MAY name any new `<layers>/<layer>` directories without restrictions except those imposed by the filesystem.
- SHOULD NOT use the `<app>` directory to store provided dependencies.

Each `/bin/build` executable of a stack buildpack:

- MUST run with root privileges.
- MUST NOT read or write to the `<app>` directory.
- MUST NOT create layers using the `<layers>/<layer>` directories.
- MAY enrich the stack layer by creating a `<layers>/stack-layer.toml`.

#### Unmet Buildpack Plan Entries

A buildpack SHOULD designate a Buildpack Plan entry as unmet if the buildpack did not satisfy the requirement described by the entry.
The lifecycle SHALL assume that all entries in the Buildpack Plan were satisfied by the buildpack unless the buildpack writes an entry with the given name to the `unmet` section of `build.toml`.

For each entry in `<plan>`:
  - **If** there is an unmet entry in `build.toml` with a matching `name`, the lifecycle
    - MUST include the entry in the `<plan>` of the next buildpack that provided an entry with that name during the detection phase.
  - **Else**, the lifecycle
    - MUST NOT include entries with matching names in the `<plan>` provided to subsequent buildpacks.

#### Bills-of-Materials

When the build is complete, a BOM (Bill-of-Materials) describing the app image MAY be generated for auditing purposes.
If generated, this BOM MUST contain all `bom` entries in each `launch.toml` at the end of each `/bin/build` execution, in adherence with the process and data format outlined in the [Platform Interface Specification](platform.md).

When the build is complete, a build BOM describing the build container MAY be generated for auditing purposes.
If generated, this build BOM MUST contain all `bom` entries in each `build.toml` at the end of each `/bin/build` execution, in adherence with the process and data format outlined in the [Platform Interface Specification](platform.md).

#### Application Layers

An application buildpack MAY create, modify, or delete `<layers>/<layer>/` directories and `<layers>/<layer>.toml` files as specified in the [Layer Types](#layer-types) section.

To decide what layer operations are appropriate, the buildpack should consider:

- Whether files in the `<app>` directory have changed since the layer was created.
- Whether the environment has changed since the layer was created.
- Whether the buildpack version has changed since the layer was created.
- Whether new application dependency versions have been made available since the layer was created.

Additionally, a buildpack MAY specify sub-paths within `<app>` as `slices` in `launch.toml`.
Separate layers MUST be created during the export phase for each slice with one or more files or directories.
This minimizes data transfer when the app directory contains a known set of files.

## Phase #4: Extend (optional)

### Purpose

The purpose of extend is to create app image layers that require changes to arbitrary locations on the filesystem.

### Process

**GIVEN:**
- The final ordered group of stack buildpacks determined during the detection phase,
- The Buildpack Plan, and
- A shell, if needed,

If the buildpack group is empty, the lifecycle MAY skip the extend phase.

For each buildpack in the group in order, the lifecycle MUST execute `/bin/build`.

1. **If** the exit status of `/bin/build` is non-zero, \
   **Then** the lifecycle MUST fail the build.

2. **If** the exit status of `/bin/build` is zero,
   1. **If** there are additional buildpacks in the group, \
      **Then** the lifecycle MUST proceed to the next buildpack's `/bin/build`.

   2. **If** there are no additional buildpacks in the group, \
      **Then** the lifecycle MUST proceed to the next phase.

For each `/bin/build` executable in each stack buildpack, the lifecycle:
- MUST provide path arguments to `/bin/build` as described in the [Buildpack Interface](#buildpack-interface) section.
- MUST configure the build environment as described in the [Environment](#environment) section.
- MUST provide all `<plan>` entries that were required by any buildpack in the group during the detection phase with names matching the names that the buildpack provided.

Correspondingly, each `/bin/build` executable:
- MUST run with root privileges.
- MUST NOT read or write to the `<app>` directory.
- MUST NOT create layers using the `<layers>/<layer>` directories.
- MAY enrich the stack layer by creating a `<layers>/stack-layer.toml`.

After each `/bin/build` executable, the lifecycle MUST:
1. **If** a corresponding `<layers>/stack-layer.toml` is present locally, \
   **Then** the lifecycle MUST
   1. Add any excluded paths to the default list of excluded paths
   2. Capture all changes made to the filesystem in non-excluded directories and store them in a snapshot.
   3. Create an independent cache snapshot containing paths identified with `cache = true` in the `<layers>/stack-layer.toml`.
2. **If** a corresponding `<layers>/stack-layer.toml` is not present locally, \
   **Then** the lifecycle MUST
   1. Capture all changes made to the filesystem in non-excluded directories and store them in a snapshot.

#### Excluded Paths

The following directories and files MUST be excluded from the stack layer created during the extend phase.

- `/tmp`
- `/cnb`
- `/layers`
- `/workspace`
- `/dev`
- `/sys`
- `/proc`
- `/var/run/secrets`
- `/etc/hostname`
- `/etc/hosts`
- `/etc/mtab`
- `/etc/resolv.conf`
- `/.dockerenv`

Changes made to these files will be ignored.

## Phase #5: Export

![Export](img/export.svg)

### Purpose

The purpose of export is to create a new OCI image using a combination of remote layers, local stack buildpack layers, local `<layers>/<layer>` layers, and the processed `<app>` directory.

### Process

**GIVEN:**
- The `<layers>` directories provided to each buildpack during the build phase,
- The `<app>` directory processed by the buildpacks during the build phase,
- The buildpack IDs associated with the buildpacks used during the build phase, in order of execution,
- A reference to the most recent version of the run image associated with the stack and mixins,
- A reference to the old OCI image processed during the analysis phase, if available, and
- A tag for a new OCI image,

**If** the run image, old OCI image, and new OCI image are not all present in the same image store, \
**Then** the lifecycle SHOULD fail the export process or inform the user that export performance is degraded.

For each stack buildpack executed during the extend phase:

1. Collect the associated snapshot generated during the extend phase and convert it to a layer.
1. Transfer the layer to the same image store as the old OCI image.
1. Ensure the absolute path are preserved in the transferred layer.
1. Collect a reference to the transferred layer.

Any cache snapshots produced by the stack buildpacks MAY be preserved for the next local build.

For each `<layers>/<layer>.toml` file that specifies `launch = true`,

1. **If** a corresponding `<layers>/<layer>` directory is present locally, \
   **Then** the lifecycle MUST
   1. Convert this directory to a layer.
   2. Transfer the layer to the same image store as the old OCI image.
   3. Ensure the absolute path of `<layers>/<layer>` is preserved in the transferred layer.
   4. Collect a reference to the transferred layer.
2. **If** a corresponding `<layers>/<layer>` directory is not present locally, \
   **Then** the lifecycle MUST
   1. Attempt to locate the corresponding layer in the old OCI image.
   2. Collect a reference to the located layer or fail export if no such layer can be found.
3. The lifecycle MUST store the `<layers>/<layer>.toml` file so that
   - It is associated with or contained within the new OCI image,
   - It is associated with the buildpack ID of the buildpack that created it, and
   - It is associated with the collected layer reference.

Next, the lifecycle MUST store `<layers>/store.toml` so that it is associated with or contained within the new OCI image.

Subsequently,

1. For `<app>`, the lifecycle MUST
   1. Convert the directory into one or more layers using `slices` in each `launch.toml` such that slices from earlier buildpacks are processed before slices from later buildpacks.
   2. Transfer the layers to the same image store as the old OCI image.
   3. Ensure all absolute paths are preserved in the transferred layer(s).
   4. Collect references to the transferred layers.
2. The lifecycle MUST construct the new OCI image such that the image is composed of
   - All new `<layers>/<layer>` filesystem layers transferred by the lifecycle,
   - All old `<layers>/<layer>` filesystem layers from the old OCI image,
   - All `<app>` filesystem layers,
   - One or more filesystem layers containing
     - The ordered buildpack IDs and
     - A combined processes list derived from all `launch.toml` files such that process types from later buildpacks override identical process types from earlier buildpacks,
   - The run image filesystem layers,
   - The executable component of the lifecycle that implements the launch phase, and
   - An `ENTRYPOINT` set to that component.

Finally, any `<layers>/<layer>` directories specified as `cache = true` in `<layers>/<layer>.toml` MAY be preserved for the next local build.
For any `<layers>/<layer>.toml` files specifying both `cache = true` and `launch = true`, the lifecycle SHOULD store a checksum of the corresponding `<layers>/<layer>` directory so that it is associated with the locally cached directory.
This allows the analysis phase to efficiently compare the locally cached layer with the corresponding old OCI image layer before the next build.

## Launch

### Purpose

The purpose of launch is to modify the running app environment using app-provided or buildpack-provided logic and start a user-provided or buildpack-provided process.

### Process

**GIVEN:**
- An OCI image exported by the lifecycle,
- A shell, if needed,

First, the lifecycle MUST locate a start command and choose an execution strategy.

To locate a start command, the lifecycle MUST follow the process outlined in the [Platform Interface Specification](platform.md).

To choose an execution strategy,

1. **If** a buildpack-provided process type is chosen as the start command,
   1. **If** the process type has `direct` set to `false`,
      1. **If** the process has one or more `args`
         **Then** the lifecycle MUST invoke a command using the shell, where `command` and each entry in `args` are shell-parsed tokens in the command.
      2. **If** the process has zero `args`
         **Then** the lifecycle MUST invoke the value of `command` as a command using the shell.

   2. **If** the process type does have `direct` set to `true`,
      **Then** the lifecycle MUST invoke the value of `command` using the `execve` syscall with values of `args` provided as arguments.

2. **If** a user-defined process type is chosen as the start command,
   **Then** the lifecycle MUST select an execution strategy as described in the [Platform Interface Specification](platform.md).

Given the start command and execution strategy,

1. The lifecycle MUST set all buildpack-provided launch environment variables as described in the [Environment](#environment) section.

2. The lifecycle MUST
   1. [execute](#execd) each file in each `<layers>/<layer>/exec.d` directory in the launch environment and set the [returned variables](#execd-output-toml) in the launch environment before continuing,
      1. Firstly, in order of `/bin/build` execution used to construct the OCI image.
      2. Secondly, in alphabetically ascending order by layer directory name.
      3. Thirdly, in alphabetically ascending order by file name.
   2. [execute](#execd) each file in each `<layers>/<layer>/exec.d/<process>` directory in the launch environment and set the [returned variables](#execd-output-toml) in the launch environment before continuing,
      1. Firstly, in order of `/bin/build` execution used to construct the OCI image.
      2. Secondly, in alphabetically ascending order by layer directory name.
      3. Thirdly, in alphabetically ascending order by file name.

3. If using an execution strategy involving a shell, the lifecycle MUST use a single shell process to
   1. source each file in each `<layers>/<layer>/profile.d` directory,
      1. Firstly, in order of `/bin/build` execution used to construct the OCI image.
      2. Secondly, in alphabetically ascending order by layer directory name.
      3. Thirdly, in alphabetically ascending order by file name.
   2. source each file in each `<layers>/<layer>/profile.d/<process>` directory,
      1. Firstly, in order of `/bin/build` execution used to construct the OCI image.
      2. Secondly, in alphabetically ascending order by layer directory name.
      3. Thirdly, in alphabetically ascending order by file name.
   3. source [†](README.md#linux-only)`<app>/.profile` or [‡](README.md#windows-only)`<app>/.profile.bat` if it is present.


3. If using an execution strategy involving a shell, the lifecycle MUST source [†](README.md#linux-only)`<app>/.profile` or [‡](README.md#windows-only)`<app>/.profile.bat` if it is present.

4. The lifecycle MUST invoke the start command with the decided execution strategy.

[†](README.md#linux-only)When executing a process using any execution strategy, the lifecycle SHOULD replace the lifecycle process in memory without forking it.

[†](README.md#linux-only)When executing a process with Bash, the lifecycle SHOULD additionally replace the Bash process in memory without forking it.

[‡](README.md#windows-only)When executing a process with Command Prompt, the lifecycle SHOULD start a new process with the same security context, terminal, working directory, STDIN/STDOUT/STDERR handles and environment variables as the Command Prompt process.

## Environment

### Provided by the Lifecycle

#### Buildpack Specific Variables

The following environment variables MUST be set by the lifecycle in each buildpack's execution environment.

These variables MAY differ between buildpacks.

| Env Variable        | Description                          | Detect | Build | Launch
|---------------------|--------------------------------------|--------|-------|--------
| `CNB_BUILDPACK_DIR` | The root of the buildpack source     | [x]    | [x]   |

#### Layer Paths

The following layer path environment variables MUST be set by the lifecycle during the build and launch phases in order to make buildpack dependencies accessible.

During the build phase, each variable designated for build MUST contain absolute paths of all previous buildpacks’ `<layers>/<layer>/` directories that are designated for build.

When the exported OCI image is launched, each variable designated for launch MUST contain absolute paths of all buildpacks’ `<layers>/<layer>/` directories that are designated for launch.

In either case,

- The lifecycle MUST order all `<layer>` paths to reflect the reversed order of the buildpack group.
- The lifecycle MUST order all `<layer>` paths provided by a given buildpack alphabetically ascending.
- The lifecycle MUST separate each path with the OS path list separator (e.g. `:` on Linux, `;` on Windows).

| Env Variable                               | Layer Path   | Contents         | Build | Launch |
|--------------------------------------------|--------------|------------------|-------|--------|
| `PATH`                                     | `/bin`       | binaries         | [x]   | [x]    |
| [†](README.md#linux-only)`LD_LIBRARY_PATH` | `/lib`       | shared libraries | [x]   | [x]    |
| [†](README.md#linux-only)`LIBRARY_PATH`    | `/lib`       | static libraries | [x]   |        |
| `CPATH`                                    | `/include`   | header files     | [x]   |        |
| `PKG_CONFIG_PATH`                          | `/pkgconfig` | pc files         | [x]   |        |


### Provided by the Platform

The following additional environment variables MUST NOT be overridden by the lifecycle.

| Env Variable    | Description                                    | Detect | Build | Launch
|-----------------|------------------------------------------------|--------|-------|--------
| `CNB_STACK_ID`  | Chosen stack ID                                | [x]    | [x]   |
| `BP_*`          | User-provided variable for buildpack           | [x]    | [x]   |
| `BPL_*`         | User-provided variable for profile.d or exec.d |        |       | [x]
| `HOME`          | Current user's home directory                  | [x]    | [x]   | [x]

During the detection and build phases, the lifecycle MUST provide any user-provided environment variables as files in `<platform>/env/` with file names and contents matching the environment variable names and contents.

When `clear-env` in `buildpack.toml` is set to `true` for a given buildpack, the lifecycle MUST NOT set user-provided environment variables in the environment of `/bin/detect` or `/bin/build`.

When `clear-env` in `buildpack.toml` is not set to `true` for a given buildpack, the lifecycle MUST set user-provided environment variables in the environment of `/bin/detect` or `/bin/build` such that:
1. For layer path environment variables, user-provided values are prepended before any existing values and are delimited by the OS path list separator.
2. For all other environment variables, user-provided values override any existing values.

Buildpacks MAY use the value of `CNB_STACK_ID` to modify their behavior when executed on different stacks.

The environment variable prefix `CNB_` is reserved.
It MUST NOT be used for environment variables that are not defined in this specification or approved extensions.

### Provided by the Buildpacks

During the build phase, buildpacks MAY write environment variable files to `<layers>/<layer>/env/`,  `<layers>/<layer>/env.build/`, and `<layers>/<layer>/env.launch/` directories.

For each `<layers>/<layer>/` designated as a build layer, for each file written to `<layers>/<layer>/env/` or `<layers>/<layer>/env.build/` by `/bin/build`, the lifecycle MUST modify an environment variable in subsequent executions of `/bin/build` according to the modification rules below.

For each file written to `<layers>/<layer>/env.launch/` by `/bin/build`, the lifecycle MUST modify an environment variable when the OCI image is launched according to the modification rules below.

#### Environment Variable Modification Rules

The lifecycle MUST consider the name of the environment variable to be the name of the file up to the first period (`.`) or to the end of the name if no periods are present.
In all cases, file contents MUST NOT be evaluated by a shell or otherwise modified before inclusion in environment variable values.

For each environment variable file the period-delimited suffix SHALL determine the modification behavior as follows.

| Suffix     | Modification Behavior
|------------|-------------------------------------------
| none       | [Override](#override)
| `.append`  | [Append](#append)
| `.default` | [Default](#default)
| `.delim`   | [Delimeter](#delimiter)
| `.override`| [Override](#override)
| `.prepend` | [Prepend](#prepend)

##### Append

The value of the environment variable MUST be a concatenation of the file contents and the contents of other files representing that environment variable.
Within that environment variable value,
- Earlier buildpacks' environment variable file contents MUST precede later buildpacks' environment variable file contents.
- Environment variable file contents originating from the same buildpack MUST be sorted alphabetically ascending by associated layer name.
- **Environment variable file contents originating in the same layer MUST be sorted such that file contents in `<layers>/<layer>/env/` precede file contents in `<layers>/<layer>/env.build/` or `<layers>/<layer>/env.launch/` which must precede file contents in `<layers>/<layer>/env.launch/<process>/`.**

##### Default

The value of the environment variable MUST only be the file contents if the environment variable is empty.
For that environment variable value,
- Earlier buildpacks' environment default variable file contents MUST override later buildpacks' environment variable file contents.
- For default environment variable file contents originating from the same buildpack, file contents that are earlier (when sorted alphabetically ascending by associated layer name) MUST override file contents that are later.
- **Default environment variable file contents originating in the same layer MUST be sorted such that file contents in `<layers>/<layer>/env/` override file contents in `<layers>/<layer>/env.build/` or `<layers>/<layer>/env.launch/` which override file contents in `<layers>/<layer>/env.launch/<process>/`.**

##### Delimiter

The file contents MUST be used to delimit any concatenation within the same layer involving that environment variable.
This delimiter MUST override the delimiters below.
If multiple operations apply to the same environment variable, all operations for a given layer containing environment variable files MUST be applied before subsequent layers are considered.

##### Override

The value of the environment variable MUST be the file contents.
For that environment variable value,
- Later buildpacks' environment variable file contents MUST override earlier buildpacks' environment variable file contents.
- For environment variable file contents originating from the same buildpack, file contents that are later (when sorted alphabetically ascending by associated layer name) MUST override file contents that are earlier.
- **Environment variable file contents originating in the same layer MUST be sorted such that file contents in `<layers>/<layer>/env.launch/<process>/` override file contents in `<layers>/<layer>/env.build/` or `<layers>/<layer>/env.launch/` which override file contents in `<layers>/<layer>/env/`.**

##### Prepend

The value of the environment variable MUST be a concatenation of the file contents and the contents of other files representing that environment variable.
Within that environment variable value,
- Later buildpacks' environment variable file contents MUST precede earlier buildpacks' environment variable file contents.
- Environment variable file contents originating from the same buildpack MUST be sorted alphabetically descending by associated layer name.
- **Environment variable file contents originating in the same layer MUST be sorted such that file contents in `<layers>/<layer>/env.launch/<process>/` precede file contents in `<layers>/<layer>/env.launch/` or `<layers>/<layer>/env.build/`, which must precede `<layers>/<layer>/env/`.**

## Security Considerations

A lifecycle may be used by a multi-tenant platform. On such a platform,

- Buildpacks may potentially be provided by both operators and users.
- OCI image storage credentials may potentially not be owned or managed by application developers.

Therefore, the following assumptions and requirements exist to prevent malicious buildpacks or applications from gaining unauthorized access to external resources.

### Assumptions of Trust

Allowed:

- The lifecycle MAY have access to credentials for reading and writing to OCI image stores.
- Buildpacks MAY have access a copy of the application source code.
- Buildpacks MAY execute application source code during the detection and build phases.

Prohibited:

- The application source code MUST NOT have access to credentials for reading and writing to OCI image stores.
- Buildpacks MUST NOT have access to credentials for reading and writing to OCI image stores.

### Requirements

The lifecycle MUST be implemented so that the detection and build phases do not have access to OCI image store credentials used in the analysis and export phases.
The lifecycle SHOULD be implemented so that each phase may run in a different container.

## Data Format

### launch.toml (TOML)

```toml
[[bom]]
name = "<dependency name>"

[bom.metadata]
# arbitrary metadata describing the dependency

[[labels]]
key = "<label key>"
value = "<label valu>"

[[processes]]
type = "<process type>"
command = "<command>"
args = ["<arguments>"]
direct = false

[[slices]]
paths = ["<app sub-path glob>"]
```

The buildpack MAY specify any number of bill-of-materials entries, labels, processes, or slices.

For each dependency contributed to the app image, the buildpack:

- SHOULD add a bill-of-materials entry to the `bom` array describing the dependency, where:
  - `name` is REQUIRED.
  - `metadata` MAY contain additional data describing the dependency.

The buildpack MAY add `bom` describing the contents of the app dir, even if they were not contributed by the buildpack.

For each label, the buildpack:

- MUST specify a `key` that is not identical to other labels provided by the same buildpack.
- MUST specify a `value` to be set in the image label.

The lifecycle MUST add each label as an image label on the created image metadata.

If multiple buildpacks define labels with the same key, the lifecycle MUST use the last label defintion ordered by buildpack execution for the image label.

For each process, the buildpack:

- MUST specify a `type`, which:
  - MUST NOT be identical to other process types provided by the same buildpack.
  - MUST only contain numbers, letters, and the characters ., _, and -.
- MUST specify a `command` that is either:
  - A command sequence that is valid when executed using the shell, if `args` is not specified.
  - A path to an executable or the file name of an executable in `$PATH`, if `args` is a list with zero or more elements.
- MAY specify an `args` list to be passed directly to the specified executable.
- MAY specify a `direct` boolean that bypasses the shell.

For each slice, buildpacks MUST specify zero or more path globs such that each path is either:

- Relative to the root of the app directory without traversing outside of the app directory.
- Absolute and contained within the app directory.

Path globs MUST:

- Follow the pattern syntax defined in the [Go standard library](https://golang.org/pkg/path/filepath/#Match).
- Match zero or more files or directories.

The lifecycle MUST process each slice as if all files matched in preceding slices no longer exists in the app directory.

The lifecycle MUST accept slices that do not contain any files or directory. However, the lifecycle MAY warn about such slices.

The lifecycle MUST include all unmatched files in the app directory in any number of additional layers in the OCI image.

### build.toml (TOML)

```toml
[[bom]]
name = "<dependency name>"

[bom.metadata]
# arbitrary metadata describing the dependency

[[unmet]]
name = "<dependency name>"
```

For each dependency contributed by the buildpack to the build environment, the buildpack:
- SHOULD add a bill-of-materials entry to the `bom` array describing the dependency, where:
  - `name` is REQUIRED.

For each unmet entry in the Buildpack Plan, the buildpack:
- SHOULD add an entry to `unmet`.

For each entry in `unmet`:
- `name` MUST match an entry in the Buildpack Plan.

### store.toml (TOML)

```toml
[metadata]
# buildpack-specific data
```

### Build Plan (TOML)

```toml
[[provides]]
name = "<dependency name>"

[[requires]]
name = "<dependency name>"
mixin = "<mixin name>"

[requires.metadata]
# buildpack-specific data

[[or]]

[[or.provides]]
name = "<dependency name>"

[[or.requires]]
name = "<dependency name>"
mixin = "<mixin name>"

[or.requires.metadata]
# buildpack-specific data

```

### Buildpack Plan (TOML)

```toml
[[entries]]
name = "<dependency name>"
mixin = "<mixin name>"

[entries.metadata]
# buildpack-specific data
```

### Layer Content Metadata (TOML)

```toml
launch = false
build = false
cache = false

[metadata]
# buildpack-specific data
```

For a given layer, the buildpack MAY specify:

- Whether the layer is cached, intended for build, and/or intended for launch.
- Metadata that describes the layer contents.

### Stack Layer Content Metadata (TOML)

```
[[excludes]]
paths = ["<sub-path glob>"]
cache = false
```

Where:

* `paths` = a list of paths to exclude from the app image layer in the extend phase. During the build phase, these paths will be removed from the filesystem before executing any application buildpacks.
* `cache` = if true, the paths will be cached even if they are removed from the filesystem.

1. Paths not referenced by an `[[excludes]]` entry will be included in the app-image layer (default).
1. Any paths with an `[[excludes]]` entry and `cache = true` will be included in the cache image, but not the app image.
1. Any paths with an `[[excludes]]` entry and `cache = false` will not be included in the cache image or the app image.

### buildpack.toml (TOML)
This section describes the 'Buildpack descriptor'.

```toml
api = "<buildpack API version>"

[buildpack]
id = "<buildpack ID>"
name = "<buildpack name>"
version = "<buildpack version>"
homepage = "<buildpack homepage>"
clear-env = false
privileged = false

[[order]]
[[order.group]]
id = "<buildpack ID>"
version = "<buildpack version>"
optional = false

[[stacks]]
id = "<stack ID>"

[stacks.requires]
mixins = [ "<mixin name>" ]

[stacks.provides]
mixins = [ "<mixin name or *>" ]

[metadata]
# buildpack-specific data
```

Where:

In the `[buildpack]` table:

* `id` - Buildpack authors MUST choose a globally unique ID, for example: "io.buildpacks.ruby".
* `privileged` - when set to `true`, requires that the lifecycle run this buildpack with root privileges.

In the `[stacks.provides]` table:

* `mixins` - a list of names that match mixins provided by this buildpack, or the `*` representing all mixins.

**The buildpack ID:**

*Key: `id = "<buildpack ID>"`*
- MUST only contain numbers, letters, and the characters `.`, `/`, and `-`.
- MUST NOT be `config` or `app`.
- MUST NOT be identical to any other buildpack ID when using a case-insensitive comparison.

The buildpack version:
- MUST be in the form `<X>.<Y>.<Z>` where `X`, `Y`, and `Z` are non-negative integers and must not contain leading zeros.
   - Each element MUST increase numerically.
   - Buildpack authors will define what changes will increment `X`, `Y`, and `Z`.

If an `order` is specified, then `stacks` MUST NOT be specified.

**The buildpack API:**

*Key: `api = "<buildpack API version>"`*
 - MUST be in form `<major>.<minor>` or `<major>`, where `<major>` is equivalent to `<major>.0`
 - MUST describe the implemented buildpack API.
 - SHOULD indicate the lowest compatible `<minor>` if buildpack behavior is consistent with multiple `<minor>` versions of a given `<major>`

#### Buildpack Implementations

A buildpack descriptor that specifies `stacks` MUST describe a buildpack that implements the [Buildpack Interface](#buildpack-interface).

Each stack in `stacks` either:
- MUST identify a compatible stack:
   - `id` MUST be set to a [valid stack ID](https://github.com/buildpacks/spec/blob/main/platform.md#stack-id).
   - `mixins` MAY contain one or more mixin names.
- Or MUST indicate compatibility with any stack:
   - `id` MUST be set to the special value `"*"`.
   - `mixins` MUST be empty.

#### Order Buildpacks

A buildpack descriptor that specifies `order` MUST be [resolvable](#order-resolution) into an ordering of buildpacks that implement the [Buildpack Interface](#buildpack-interface).

A buildpack reference inside of a `group` MUST contain an `id` and `version`.

### Exec.d Output (TOML)
```
<name> = "<value>"
```

The output from an `exec.d` script MAY contain any number of top-level key/value pairs.

Each `name`:
* MUST be a [bare key](https://github.com/toml-lang/toml/blob/master/toml.md#keys).
* MUST be a valid environment variable name on the runtime operating system.

Each `key`:
* MUST be a [basic string](https://github.com/toml-lang/toml/blob/master/toml.md#string).

## Deprecations
This section describes all the features that are deprecated.

### `0.3`

#### Build Plan (TOML) `requires.version` Key

The `requires.version` and `or.requires.version` keys are deprecated.

```toml
[[requires]]
name = "<dependency name>"
version = "<dependency version>"

[[or.requires]]
name = "<dependency name>"
version = "dependency version>"
```

To upgrade, buildpack authors SHOULD set `requires.version` as `requires.metadata.version` and `or.requires.version` as `or.requires.metadata.version`.

```toml
[[requires]]
name = "<dependency name>"

[requires.metadata]
version = "<dependency version>"

[[or.requires]]
name = "<dependency name>"

[or.requires.metadata]
version = "<dependency version>"
```

If `requires.version` and `requires.metadata.version` or `or.requires.version` and `or.requires.metadata.version` are both defined then lifecycle will fail.

For backwards compatibility, the lifecycle will produce a Buildpack Plan (TOML) that puts `version` in `entries.metadata` as long as `version` does not exist in `requires.metadata`.

```toml
[[entries]]
name = "<dependency name>"

[entries.metadata]
version = "<dependency version>"
```
