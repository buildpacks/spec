# Buildpack Interface Specification

This document specifies the interface between a single lifecycle and one or more buildpacks.

A lifecycle is a program that uses buildpacks to transform application source code into an OCI image containing the compiled application.

This is accomplished in four phases:

1. **Detection,** where an optimal selection of compatible buildpacks is chosen.
2. **Analysis,** where metadata about OCI layers generated during a previous build are made available to buildpacks.
3. **Build,** where buildpacks use that metadata to generate only the OCI layers that need to be replaced.
4. **Export,** where the remote layers are replaced by the generated layers.

The `ENTRYPOINT` of the OCI image contains logic implemented by the lifecycle that executes during the **Launch** phase.

Additionally, a lifecycle can use buildpacks to create a containerized environment for developing or testing application source code.

This is accomplished in two phases:

1. **Detection,** where an optimal selection of compatible buildpacks is chosen.
2. **Development,** where the lifecycle uses those buildpacks to create a containerized development environment.

## Table of Contents

1. [Buildpack Interface](#buildpack-interface)
   1. [Key](#key)
   2. [Detection](#detection)
   3. [Build](#build)
   4. [Development](#development)
2. [App Interface](#app-interface)
3. [Phase #1: Detection](#phase-1-detection)
4. [Phase #2: Analysis](#phase-2-analysis)
5. [Phase #3: Build](#phase-3-build)
6. [Phase #4: Export](#phase-4-export)
7. [Launch](#launch)
8. [Development Setup](#development-setup)
9. [Environment](#environment)
   1. [Provided by the Lifecycle](#provided-by-the-lifecycle)
   2. [Provided by the Platform](#provided-by-the-platform)
   3. [Provided by the Buildpacks](#provided-by-the-buildpacks)
10. [Security Considerations](#security-considerations)
    1. [Assumptions of Trust](#assumptions-of-trust)
    2. [Requirements](#requirements)
11. [Artifact Format](#artifact-format)
    1. [Buildpack Package](#buildpack-package)
12. [Data Format](#data-format)
    1. [buildpack.toml (TOML)](#buildpack.toml-(toml))
    2. [launch.toml (TOML)](#launch.toml-(toml))
    3. [develop.toml (TOML)](#develop.toml-(toml))
    4. [Build Plan (TOML)](#build-plan-(toml))

## Buildpack Interface

The following specifies the interface implemented by executables in each buildpack.
The lifecycle MUST invoke these executables as described in the Phase sections.

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

Executable: `/bin/detect`, Working Dir: `<app[AR]>`

| Input         | Description
|---------------|----------------------------------------------
| `/dev/stdin`  | Merged plan from previous detections (TOML)

| Output        | Description
|---------------|----------------------------------------------
| [exit status] | Pass (0), fail (100), or error (1-99, 101+)
| `/dev/stdout` | Updated plan for subsequent detections (TOML)
| `/dev/stderr` | Detection logs (all)

###  Build

Executable: `/bin/build <platform[AR]> <cache[EC]> <launch[EI]>`, Working Dir: `<app[AI]>`

| Input                         | Description
|-------------------------------|----------------------------------------------
| `/dev/stdin`                  | Build plan from detection (TOML)
| `<platform>/env/`             | User-provided environment variables for build
| `<platform>/#`                | Platform-specific extensions

| Output                        | Description
|-------------------------------|----------------------------------------------
| [exit status]                 | Success (0) or failure (1+)
| `/dev/stdout`                 | Logs (info)
| `/dev/stderr`                 | Logs (warnings, errors)
| `<cache>/<layer>/bin/`        | Binaries for subsequent buildpacks
| `<cache>/<layer>/lib/`        | Libraries for subsequent buildpacks
| `<cache>/<layer>/include/`    | C/C++ headers for subsequent buildpacks
| `<cache>/<layer>/pkgconfig/`  | Search path for pkg-config
| `<cache>/<layer>/env/`        | Env vars for subsequent buildpacks
| `<cache>/<layer>/*`           | Other cached content
| `<launch>/launch.toml`        | Launch metadata (see [launch.toml](#launch.toml-(toml)))
| `<launch>/<layer>.toml`       | Layer content metadata
| `<launch>/<layer>/bin/`       | Binaries for launch
| `<launch>/<layer>/lib/`       | Shared libraries for launch
| `<launch>/<layer>/profile.d/` | Scripts sourced by bash before launch
| `<launch>/<layer>/*`          | Other content for launch

### Development

Executable: `/bin/develop <platform[A]> <cache[EC]>`, Working Dir: `<app[A]>`

| Input                        | Description
|------------------------------|----------------------------------------------
| `/dev/stdin`                 | Build plan from detection (TOML)
| `<platform>/env/`            | User-provided environment variables for build
| `<platform>/#`               | Platform-specific extensions

| Output                       | Description
|------------------------------|----------------------------------------------
| [exit status]                | Success (0) or failure (1+)
| `/dev/stdout`                | Logs (info)
| `/dev/stderr`                | Logs (warnings, errors)
| `<cache>/develop.toml`       | Development metadata (see [develop.toml](#develop.toml-(toml)))
| `<cache>/<layer>/bin/`       | Binaries for subsequent buildpacks & app
| `<cache>/<layer>/lib/`       | Libraries for subsequent buildpacks & app
| `<cache>/<layer>/include/`   | C/C++ headers for subsequent buildpacks
| `<cache>/<layer>/pkgconfig/` | Search path for pkg-config
| `<cache>/<layer>/env/`       | Env vars for subsequent buildpacks & app
| `<cache>/<layer>/*`          | Other content for subsequent buildpacks & app


## App Interface

| Output           | Description
|------------------|----------------------------------------------
| `<app>/.profile` | Script sourced by bash before launch

## Phase #1: Detection

![Detection](img/detection.svg)

### Purpose

The purpose of detection is to find an ordered group of buildpacks to use during the build phase.
These buildpacks must be compatible with the app.

### Process

**GIVEN:**
- An ordered list of ordered buildpack groups and
- A directory containing application source code,

For each buildpack in each group in order, the lifecycle MUST execute `/bin/detect`.

1. **IF** the exit status of `/bin/detect` is non-zero and the buildpack is not marked optional, \
   **THEN** the lifecycle MUST proceed to the next group or fail detection completely if no more groups are present.

2. **IF** the exit status of `/bin/detect` is zero or the buildpack is marked optional,
   1. **IF** the buildpack is not the last buildpack in the group, \
      **THEN** the lifecycle MUST proceed to the next buildpack in the group.

   2. **IF** the buildpack is the last buildpack in the group,
      1. **IF** no exit statuses from `/bin/detect` in the group are zero \
         **THEN** the lifecycle MUST proceed to the next group or fail detection completely if no more groups are present.

      2. **IF** at least one exit status from `/bin/detect` in the group is zero \
         **THEN** the lifecycle MUST select this group and proceed to the analysis phase.

The selected group MUST be filtered to only include buildpacks with exit status zero.
The order of the buildpacks in the group MUST otherwise be preserved.

The `/bin/detect` executable in each buildpack, when executed:

1. MAY examine the app directory and environment variables.
2. MAY emit error, warning, or debug messages to `stderr`.
3. MAY receive a TOML-formatted map called a Build Plan on `stdin`.
4. MAY output changes to the Build Plan on `stdout`.
5. MUST set an exit status code as described in the [Buildpack Interface](#buildpack-interface) section.

For each `/bin/detect`, the Build Plan received on `stdin` MUST be a combined map derived from the output of all previous `/bin/detect` executables.
The lifecycle MUST construct this map such that the top-level values from later buildpacks override the entire top-level values from earlier buildpacks.
The lifecycle MUST NOT include any changes in this map that are output by optional buildpacks that returned non-zero exit statuses.
The final Build Plan is the complete combined map that includes the output of the final `/bin/detect` executable.

The lifecycle MAY execute each `/bin/detect` within a group in parallel.
Therefore, reading from `stdin` in `/bin/detect` MUST block until the previous `/bin/detect` in the group closes `stdout`.

The lifecycle MUST run `/bin/detect` for all buildpacks in a group in a container using common stack with a common set of additional operating system packages.
The lifecycle MUST fail detection if any of those buildpacks does not list that stack in `buildpack.toml`.
The lifecycle MUST fail detection if any of those buildpacks specifies an additional operating system package associated with that stack in `buildpack.toml` that is unavailable in the container.

## Phase #2: Analysis

![Analysis](img/analysis.svg)

### Purpose

The purpose of analysis is to retrieve `<launch>/<layer>.toml` files that buildpacks may use to optimize the build and export phases.

Each `<launch>/<layer>.toml` file represents a remote filesystem layer that the buildpack may keep, replace, or remove during the build phase.

### Process

The lifecycle SHOULD attempt to locate a reference to an OCI image from a previous build that:

- Was created using some version of the same application source code.
- Is readable by the lifecycle.
- Was created using the lifecycle.
- Is as recent as possible.

The lifecycle MUST skip analysis and proceed to the build phase if no such image can be located.

**GIVEN:**
- A reference to the previously created OCI image described above and
- The final ordered group of buildpacks determined during detection,

For each buildpack in the group,

1. Any `<launch>/<layer>.toml` files that were present at the end of the build of the previously created OCI image are retrieved.
2. Those `<layer>.toml` files are placed on the filesystem so that they appear in the buildpack's `<launch>/` directory during the build phase.

The lifecycle MUST NOT download any filesystem layers from the previous OCI image.

After analysis, the lifecycle MUST proceed to the build phase.

## Phase #3: Build

![Build](img/build.svg)

### Purpose

The purpose of build is to transform application source code into runnable artifacts that can be packaged into a container.

During the build phase, typical buildpacks might:

1. Provide the application with dependencies for launch in `<launch>/<layer>`.
2. Provide subsequent buildpacks with dependencies in `<cache>/<layer>`.
3. Compile the application source code into object code.
4. Remove application source code that is not necessary for launch.
5. Provide start command in `<launch>/launch.toml`.

The purpose of separate `<layer>` directories is to:

1. Minimize the execution time of the build phase.
2. Minimize persistent disk usage.

This is achieved by:

1. Reducing the number of necessary build operations during the build phase.
2. Reducing data transfer during the export phase.
3. Enabling de-duplication of stored image layers.

### Process

**GIVEN:**
- The final ordered group of buildpacks determined during detection,
- A directory containing application source code,
- The final Build Plan,
- Any `<launch>/<layer>.toml` files placed on the filesystem during the analysis phase, and
- The most recent local `<cache>` directories from a build of a version of the application source code,

For each buildpack in the group in order, the lifecycle MUST execute `/bin/build`.

1. **IF** the exit status of `/bin/build` is non-zero, \
   **THEN** the lifecycle MUST fail the build.

2. **IF** the exit status of `/bin/build` is zero,
   1. **IF** there are additional buildpacks in the group, \
      **THEN** the lifecycle MUST proceed to the next buildpack's `/bin/build`.

   2. **IF** there are no additional buildpacks in the group, \
      **THEN** the lifecycle MUST proceed to the export phase.

For each `/bin/build` executable in each buildpack, the lifecycle:

- MUST provide a Build Plan to `stdin` of `/bin/build`.
- MUST configure the build environment as defined in the [Environment](#environment) section.
- MUST provide path arguments to `/bin/build` as defined in the [Buildpack Interface](#buildpack-interface) section.
- MAY provide an empty `<cache>` directory if the platform does not make it available.

Correspondingly, each `/bin/build` executable:

- MAY read or write to the `<app>` directory.
- MAY read a Build Plan from `stdin`.
- MAY write a list of possible commands for launch to `<launch>/launch.toml` in `launch.toml` format.
- MAY supply dependencies in `<cache>/<layer>` directories.
- MAY supply dependencies in new `<launch>/<layer>` directories and create corresponding `<launch>/<layer>.toml` files.
- MAY name any new `<layer>` directories without restrictions except those imposed by the filesystem.
- SHOULD NOT use the `<app>` directory to store provided dependencies.
- MUST NOT provide paths for `<launch>/<layer>` directories to other buildpacks.

The buildpack SHOULD use the contents of a given pre-existing `<launch>/<layer>.toml` file to decide:

1. Whether to create a `<launch>/<layer>` directory that will replace a remote layer during the export phase.
2. Whether to remove the `<launch>/<layer>.toml` file to delete a remote layer during the export phase.
3. Whether to modify the `<launch>/<layer>.toml` file for the next build.

To make this decision, the buildpack should consider:

1. Whether files in the `<app>` directory are sufficiently similar to the previous build.
2. Whether the environment is sufficiently similar to the previous build.
3. Whether files in `<cache>` directories are sufficiently similar to the previous build.
4. Whether the buildpack version has changed since the previous build.

## Phase #4: Export

![Export](img/export.svg)

### Purpose

The purpose of export is to create an new OCI image using a combination of remote layers, local `<launch>/<layer>` layers, and the processed `<app>` directory.

### Process

**GIVEN:**
- The `<launch>` directories provided to each buildpack during the build phase,
- The `<app>` directory processed by the buildpacks during the build phase,
- The buildpack IDs associated with the buildpacks used during the build phase, in order of execution,
- A reference to the most recent version of the run image associated with the stack and additional operating system packages,
- A reference to the old OCI image associated with the `<launch>/<layer>.toml` files that were retrieved during the analysis phase, and
- A tag for a new OCI image,

**IF** the run image, old OCI image, and new OCI image are not all present in the same image store, \
**THEN** the lifecycle SHOULD fail the export process or inform the user that export performance is degraded.

For each `<launch>/<layer>.toml` file,

1. **IF** a corresponding `<launch>/<layer>` directory is present locally, \
   **THEN** the lifecycle MUST
   1. Convert this directory to a layer.
   2. Transfer the layer to the same image store as the old OCI image.
   3. Ensure the absolute path of `<launch>/<layer>` is preserved in the transferred layer.
   4. Collect a reference to the transferred layer.
2. **IF** a corresponding `<launch>/<layer>` directory is not present locally, \
   **THEN** the lifecycle MUST
   1. Attempt to locate the corresponding layer in the old OCI image.
   2. Collect a reference to the located layer or fail export if no such layer can be found.
3. The lifecycle MUST store the `<launch>/<layer>.toml` file so that
   - It is associated with or contained within new OCI image,
   - It is associated with the buildpack ID of the buildpack that created it, and
   - It is associated with the collected layer reference.

Subsequently,

1. For `<app>`, the lifecycle MUST
   1. Convert the directory into one or more layers,
   2. Transfer the layers to the same image store as the old OCI image.
   3. Ensure absolute path of the directory is preserved in the transferred layer(s).
   4. Collect references to the transferred layers.
2. The lifecycle MUST construct the new OCI image such that the image is composed of
   - All new `<launch>/<layer>` filesystem layers transferred by the lifecycle,
   - All old `<launch>/<layer>` filesystem layers from the old OCI image,
   - All `<app>` filesystem layers,
   - One or more filesystem layers containing
     - The ordered buildpack IDs and
     - A combined processes list derived from all `launch.toml` files such that process types from later buildpacks override identical process types from earlier buildpacks,
   - The run image filesystem layers, and
   - An `ENTRYPOINT` set to an executable component of the lifecycle that implements the launch phase.

The lifecycle MUST NOT access the contents of any filesystem layers from the previous OCI image.

## Launch

### Purpose

The purpose of launch is to modify the running app environment using app-provided or buildpack-provided logic and start a user-provided or buildpack-provided process.

### Process

**GIVEN:**
- An OCI image exported by the lifecycle,
- An optional process type specified by `PACK_PROCESS_TYPE`, and
- Bash version 3 or greater,

When the OCI image is launched,

1. The lifecycle MUST source each file in each `<launch>/<layer>/profile.d` directory using the same Bash shell process,
   1. Firstly, in order of `/bin/build` execution used to construct the OCI image.
   2. Secondly, in alphabetically ascending order by layer directory name.
   3. Thirdly, in alphabetically ascending order by file name.

2. In the Bash shell process used to source the `profile.d` scripts, the lifecycle MUST source `<app>/.profile` if it is present.

3. If `CMD` in the container configuration is not empty, the lifecycle MUST join each argument with a space and execute the resulting command in the container using the Bash shell process used to source the `profile.d` scripts.

4. If `CMD` in the container configuration is empty,
   1. **IF** the `PACK_PROCESS_TYPE` environment variable is set,
      1. **IF** the value of `PACK_PROCESS_TYPE` corresponds to a process in `<launch>/launch.toml`, \
         **THEN** the lifecycle MUST execute the corresponding command in the container using the Bash shell process used to source the `profile.d` scripts.

      2. **IF** the value of `PACK_PROCESS_TYPE` does not correspond to a process in `<launch>/launch.toml`, \
         **THEN** launch fails.

   2. **IF** the `PACK_PROCESS_TYPE` environment variable is not set,
      1. **IF** there is a process with a `web` process type in `<launch>/launch.toml`, \
         **THEN** the lifecycle MUST execute the corresponding command in the container using the Bash shell process used to source the `profile.d` scripts.

      2. **IF** there is not a process with a `web` process type in `<launch>/launch.toml`, \
         **THEN** launch fails.

When executing a process with Bash, the lifecycle SHOULD replace the Bash process in memory with the resulting command process if possible.

## Development Setup

### Purpose

The purpose of development setup is to create a containerized environment for developing or testing application source code.

During the development setup phase, typical buildpacks might:

1. Provide the app as well as subsequent buildpacks with dependencies in `<cache>/<layer>`.
2. Provide a command to start a development server in `<cache>/develop.toml`.
2. Provide a command to run a test suite in `<cache>/develop.toml`.

### Process

**GIVEN:**
- The final ordered group of buildpacks determined during detection,
- A directory containing application source code,
- The final Build Plan,
- The most recent local `<cache>` directories from a development setup of a version of the application source code, and
- Bash version 3 or greater,

For each buildpack in the group in order, the lifecycle MUST execute `/bin/develop`.

1. **IF** the exit status of `/bin/develop` is non-zero, \
   **THEN** the lifecycle MUST fail the development setup.

2. **IF** the exit status of `/bin/develop` is zero,
   1. **IF** there are additional buildpacks in the group, \
      **THEN** the lifecycle MUST proceed to the next buildpack's `/bin/develop`.

   2. **IF** there are no additional buildpacks in the group, \
      **THEN** the lifecycle MUST launch the process type specified by `PACK_PROCESS_TYPE`.

For each `/bin/develop` executable in each buildpack, the lifecycle:

- MUST provide a Build Plan to `stdin` of `/bin/develop`.
- MUST configure the build environment as defined in the [Environment](#environment) section.
- MUST provide path arguments to `/bin/develop` as defined in the [Buildpack Interface](#buildpack-interface) section.
- MAY provide an empty `<cache>` directory if the platform does not make it available.

Correspondingly, each `/bin/develop` executable:

- MAY read from the app directory.
- MAY write files to the app directory in an idempotent manner.
- MAY read a Build Plan from `stdin`.
- MAY supply dependencies in `<cache>/<layer>` directories.
- MAY name any new `<layer>` directories without restrictions except those imposed by the filesystem.
- SHOULD NOT use the `<app>` directory to store provided dependencies.

After the last `/bin/develop` finishes executing,

1. **IF** the `PACK_PROCESS_TYPE` environment variable is set,
   1. **IF** the value of `PACK_PROCESS_TYPE` corresponds to a process in `<cache>/develop.toml`, \
      **THEN** the lifecycle MUST execute the corresponding command in the container using Bash.

   2. **IF** the value of `PACK_PROCESS_TYPE` does not correspond to a process in `<cache>/develop.toml`, \
      **THEN** the lifecycle MUST fail development setup.

2. **IF** the `PACK_PROCESS_TYPE` environment variable is not set,
   1. **IF** there is a process with a `web` process type in `<cache>/develop.toml`, \
      **THEN** the lifecycle MUST execute the corresponding command in the container using Bash.

   2. **IF** there is not a process with a `web` process type in `<cache>/develop.toml`, \
      **THEN** the lifecycle MUST fail development setup.

When executing a process with Bash, the lifecycle SHOULD replace the Bash process in memory with the resulting command process if possible.

## Environment

### Provided by the Lifecycle

The following environment variables MUST be set by the lifecycle in order to make buildpack dependencies accessible.

During the build phase, each variable designated for build MUST contain absolute paths of all previous buildpacks’ `<cache>/<layer>/` directories.

When the exported OCI image is launched, each variable designated for launch MUST contain absolute paths of all buildpacks’ `<launch>/<layer>/` directories.

In either case,

- The lifecycle MUST order all `<layer>` paths to reflect the order of the buildpack group.
- The lifecycle MUST order all `<layer>` paths provided by a given buildpack alphabetically ascending.
- The lifecycle MUST separate each path with the OS path list separator (e.g., `:` on Linux).

| Env Variable      | Layer Path   | Contents         | Build | Launch
|-------------------|--------------|------------------|-------|--------
| `PATH`            | `/bin`       | binaries         | [x]   | [x]
| `LD_LIBRARY_PATH` | `/lib`       | shared libraries | [x]   | [x]
| `LIBRARY_PATH`    | `/lib`       | static libraries | [x]   |
| `CPATH`           | `/include`   | header files     | [x]   |
| `PKG_CONFIG_PATH` | `/pkgconfig` | pc files         | [x]   |

### Provided by the Platform

The following additional environment variables MUST NOT be overridden by the lifecycle.

| Env Variable    | Description                            | Detect | Build | Launch
|-----------------|----------------------------------------|--------|-------|--------
| `PACK_STACK_ID` | Chosen stack ID                        | [x]    | [x]   |
| `BP_*`          | User-specified variable for buildpack  | [x]    | [x]   |
| `BPL_*`         | User-specified variable for profile.d  |        |       | [x]
| `HOME`          | Current user's home directory          | [x]    | [x]   | [x]

The lifecycle MUST provide any user-provided environment variables as files in `<platform>/env/` with file names and contents matching the environment variable names and contents.

The lifecycle MUST NOT set user-provided environment variables in the environment of `/bin/build` directly.

Buildpacks MAY use the value of `PACK_STACK_ID` to modify their behavior when executed on different stacks.

The environment variable prefix `PACK_` is reserved.
It MUST NOT be used for environment variables that are not defined in this specification or approved extensions.

### Provided by the Buildpacks

During the build phase, buildpacks MAY write environment variable files to `<cache>/<layer>/env/` directories.

For each file written to `<cache>/<layer>/env/` by `/bin/build`, the lifecycle MUST modify an environment variable in subsequent executions of `/bin/build`.
The lifecycle MUST set the name of the environment variable to the name of the file up to the first period (`.`) or to the end of the name if no periods are present.

If the environment variable has no period-delimited suffix, then the value of the environment variable MUST be a concatenation of the file contents and the contents of other identically named files in other `<cache>/<layer>/env/` directories delimited by the OS path list separator.
Within that environment variable value,
- Earlier buildpacks' environment variable file contents MUST precede later buildpacks' environment variable file contents.
- Environment variable file contents originating from the same buildpack MUST be sorted alphabetically ascending by associated layer name.

If the environment variable file name ends in `.append`, then the value of the environment variable MUST be a concatenation of the file contents and the contents of other identically named files in other `<cache>/<layer>/env/` directories without any delimitation.
Within that environment variable value,
- Earlier buildpacks' environment variable file contents MUST precede later buildpacks' environment variable file contents.
- Environment variable file contents originating from the same buildpack MUST be sorted alphabetically ascending by associated layer name.

If the environment variable file name ends in `.override`, then the value of the environment variable MUST be the file contents or the contents of another identically named file in another `<cache>/<layer>/env/` directory.
For that environment variable value
- Later buildpacks' environment variable file contents MUST override earlier buildpacks' environment variable file contents.
- For environment variable file contents originating from the same buildpack, file contents that are later (when sorted alphabetically ascending by associated layer name) MUST override file contents that are earlier.

In all cases, file contents MUST NOT be evaluated by a shell or otherwise modified before inclusion in environment variable values.

## Security Considerations

A lifecycle may be used by a multi-tenant platform. On such a platform,

- Buildpacks may be provided by both operators and users.
- OCI image storage credentials may not be owned or managed by application developers.

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

## Artifact Format

### Buildpack Package

Buildpacks SHOULD be packaged as gzip-compressed tarballs with extension `.tgz`.

Buildpacks MUST:
- Contain `/buildpack.toml` and `/bin/detect`.
- Contain one or both of `/bin/build` and `/bin/develop`.

## Data Format

### buildpack.toml (TOML)

```toml
[buildpack]
id = "<buildpack ID>"
name = "<buildpack name>"
version = "<buildpack version>"

[[stacks]]
id = "<stack ID>"
build-packages = ["<package name>"]
run-packages = ["<package name>"]
build-images = ["<build image tag>"]
run-images = ["<run image tag>"]

[metadata]
# buildpack-specific data
```

Buildpack authors MUST choose a globally unique ID, for example: "io.buildpacks.ruby".

The buildpack ID:
- MUST only contain numbers, letters, and the charactors `.`, `/`, and `-`.
- MUST NOT be `config` or `app`.
- MUST NOT be identical to any other buildpack ID when using a case-insensitive comparison.

Stack authors MUST choose a globally unique ID, for example: "io.buildpacks.mystack".

The stack ID:
- MUST only contain numbers, letters, and the charactors `.`, `/`, and `-`.
- MUST NOT be identical to any other stack ID when using a case-insensitive comparison.

The stack `build-images` and `run-images` are suggested sources of images for platforms that are unaware of the stack ID.
Buildpack authors MUST ensure that build images include all packages specified in `build-packages`.
Buildpack authors MUST ensure that run images include all packages specified in `run-packages`.

### launch.toml (TOML)

```toml
[[processes]]
type = "<process type>"
command = "<command>"
```

Buildpacks MUST specify:

- A unique process type for each entry within a `launch.toml` file.
- A command that is valid when executed using the Bash 3+ shell.

### develop.toml (TOML)

```toml
[[processes]]
type = "<process type>"
command = "<command>"
```

Buildpacks MUST specify:

- A unique process type for each entry within a `develop.toml` file.
- A command that is valid when executed using the Bash 3+ shell.

### Build Plan (TOML)

```toml
[<dependency name>]
version = "<dependency version>"
provider = "<buildpack ID>"

[<dependency name>.metadata]
# buildpack-specific data
```

For a given dependency, the buildpack MAY specify:

- The dependency version. If a version range is needed, semver notation SHOULD be used to specify the range.
- The ID of the buildpack that will provide the dependency.
