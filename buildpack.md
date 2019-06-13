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
   5. [Layer Types](#layer-types)
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
    1. [buildpack.toml (TOML)](#buildpack.toml-toml)
    2. [launch.toml (TOML)](#launch.toml-toml)
    4. [Build Plan (TOML)](#build-plan-toml)
    5. [Layer Content Metadata (TOML)](#layer-content-metadata-toml)

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

Executable: `/bin/detect <platform[AR]> <plan[E]>`, Working Dir: `<app[AR]>`

| Input             | Description
|-------------------|----------------------------------------------
| `$0`              | Absolute path of `/bin/detect` executable
| `/dev/stdin`      | Merged build plan from previous detections (TOML)
| `<platform>/env/` | User-provided environment variables for build
| `<platform>/#`    | Platform-specific extensions

| Output             | Description
|--------------------|----------------------------------------------
| [exit status]      | Pass (0), fail (100), or error (1-99, 101+)
| `/dev/stdout`      | Logs (info)
| `/dev/stderr`      | Logs (warnings, errors)
| `<plan>`           | Contributions to the the build plan (TOML)


###  Build

Executable: `/bin/build <layers[EIC]> <platform[AR]> <plan[E]>`, Working Dir: `<app[AI]>`

| Input             | Description
|-------------------|----------------------------------------------
| `$0`              | Absolute path of `/bin/build` executable
| `/dev/stdin`      | Build plan from detection (TOML)
| `<platform>/env/` | User-provided environment variables for build
| `<platform>/#`    | Platform-specific extensions

| Output                         | Description
|--------------------------------|-----------------------------------------------
| [exit status]                  | Success (0) or failure (1+)
| `/dev/stdout`                  | Logs (info)
| `/dev/stderr`                  | Logs (warnings, errors)
| `<plan>`                       | Claims of contributions to the build plan (TOML)
| `<layers>/launch.toml`         | App metadata (see [launch.toml](#launch.toml-toml))
| `<layers>/store.toml`          | Persistent metadata (see [store.toml](#store.toml-toml))
| `<layers>/<layer>.toml`        | Layer metadata (see [Layer Content Metadata](#layer-content-metadata-toml))
| `<layers>/<layer>/bin/`        | Binaries for launch and/or subsequent buildpacks
| `<layers>/<layer>/lib/`        | Shared libraries for launch and/or subsequent buildpacks
| `<layers>/<layer>/profile.d/`  | Scripts sourced by Bash before launch
| `<layers>/<layer>/include/`    | C/C++ headers for subsequent buildpacks
| `<layers>/<layer>/pkgconfig/`  | Search path for pkg-config for subsequent buildpacks
| `<layers>/<layer>/env/`        | Env vars for launch and/or subsequent buildpacks
| `<layers>/<layer>/env.launch/` | Env vars for launch (after `env`, before `profile.d`)
| `<layers>/<layer>/env.build/`  | Env vars for subsequent buildpacks (after `env`)
| `<layers>/<layer>/*`           | Other content for launch and/or subsequent buildpacks

### Development

Executable: `/bin/develop <layers[EC]> <platform[AR]> <plan[E]>`, Working Dir: `<app[A]>`

| Input             | Description
|-------------------|----------------------------------------------
| `$0`              | Absolute path of `/bin/detect` executable
| `/dev/stdin`      | Build plan from detection (TOML)
| `<platform>/env/` | User-provided environment variables for build
| `<platform>/#`    | Platform-specific extensions

| Output                         | Description
|--------------------------------|----------------------------------------------
| [exit status]                  | Success (0) or failure (1+)
| `/dev/stdout`                  | Logs (info)
| `/dev/stderr`                  | Logs (warnings, errors)
| `<plan>`                       | Claims of contributions to the build plan (TOML)
| `<layers>/launch.toml`         | App metadata (see [launch.toml](#launch.toml-toml))
| `<layers>/store.toml`          | Persistent metadata (see [store.toml](#store.toml-toml))
| `<layers>/<layer>.toml`        | Layer metadata (see [Layer Content Metadata](#layer-content-metadata-toml))
| `<layers>/<layer>/bin/`        | Binaries for launch and/or subsequent buildpacks
| `<layers>/<layer>/lib/`        | Shared libraries for launch and/or subsequent buildpacks
| `<layers>/<layer>/profile.d/`  | Scripts sourced by Bash before launch
| `<layers>/<layer>/include/`    | C/C++ headers for subsequent buildpacks
| `<layers>/<layer>/pkgconfig/`  | Search path for pkg-config for subsequent buildpacks
| `<layers>/<layer>/env/`        | Env vars for launch and/or subsequent buildpacks
| `<layers>/<layer>/env.launch/` | Env vars for launch (after `env`, before `profile.d`)
| `<layers>/<layer>/env.build/`  | Env vars for subsequent buildpacks (after `env`)
| `<layers>/<layer>/*`           | Other content for launch and/or subsequent buildpacks

### Layer Types

Using the [Layer Content Metadata](#layer-content-metadata-toml) provided by a buildpack in a `<layers>/<layer>.toml` file, the lifecycle MUST determine:

- Whether the layer directory in `<layers>/<layer>/` should be available to the app (via the `launch` boolean).
- Whether the layer directory in `<layers>/<layer>/` should be available to subsequent buildpacks (via the `build` boolean).
- Whether and how the layer directory in `<layers>/<layer>/` should be persisted to subsequent builds of the same OCI image (via the `cache` boolean).

This section does not apply to the Development Setup phase, which does not generate an OCI image.

#### Launch Layers

A buildpack MAY specify that a `<layers>/<layer>/` directory is a launch layer by placing `launch = true` in `<layers>/<layer>.toml`.

The lifecycle MUST make all launch layers accessible to the app as defined in the [Environment](#environment) section.

The lifecycle MUST include each launch layer in the built OCI image.
The lifecycle MUST also store the Layer Content Metadata associated with each layer so that it can be recovered using the layer Diff ID.

Before a given re-build:
- If a launch layer is marked `cache = false`, the lifecycle:
  - MUST restore the entire `<layers>/<layer>.toml` file from the previous build to the same path and
  - MUST NOT restore the corresponding `<layers>/<layer>/` directory from any previous build.
- If a launch layer is marked `cache = true`, the lifecycle:
  - MUST either restore the entire `<layers>/<layer>.toml` file and corresponding `<layers>/<layer>/` directory from the previous build to the same paths or
  - MUST restore neither the `<layers>/<layer>.toml` file nor corresponding `<layers>/<layer>/` directory.

After a given re-build:
- If a buildpack keeps `launch = true` in `<layers>/<layer>.toml` and leaves no `<layers>/<layer>/` directory, the lifecycle:
  - MUST keep the corresponding layer from the previous build in the OCI image and
  - MUST replace the `<layers>/<layer>.toml` in the OCI image with the version present after the re-build.
- If a buildpack keeps `launch = true` in `<layers>/<layer>.toml` and leaves a `<layers>/<layer>/` directory, the lifecycle:
  - MUST replace the corresponding layer in the OCI image with the directory contents present after the re-build and
  - MUST replace the `<layers>/<layer>.toml` in the OCI image with the version present after the re-build.
- If a buildpack removes `launch = true` from `<layers>/<layer>.toml` or deletes `<layers>/<layer>.toml`, then the lifecycle MUST NOT include any corresponding layer in the OCI image.

#### Build Layers

A buildpack MAY specify that a `<layers>/<layer>/` directory is a build layer by placing `build = true` in `<layers>/<layer>.toml`.

The lifecycle MUST make all build layers accessible to subsequent buildpacks as defined in the [Environment](#environment) section.

Before the next re-build:
- If the layer is marked `cache = true`, the lifecycle MAY restore the `<layers>/<layer>/` directory and Layer Content Metadata from any previous build to the same path.
- If the layer is marked `cache = false`, the lifecycle MUST NOT restore the `<layers>/<layer>/` directory or the Layer Content Metadata from any previous build.

#### Other Layers

For layers marked `launch = true` and `build = true`, the most strict requirements of each type apply.

Therefore, the lifecycle MUST consider such layers to be launch layers that are also accessible to subsequent buildpacks as defined in the [Environment](#environment) section.

The lifecycle MUST consider layers that are marked `launch = false` and `build = false` to be build layers that are not accessible to subsequent buildpacks.

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

- MAY read the app directory.
- MAY read the detect environment as defined in the [Environment](#environment) section.
- MAY emit error, warning, or debug messages to `stderr`.
- MAY receive a TOML-formatted [Build Plan](#build-plan-toml) on `stdin`.
- MAY contribute to the Build Plan by writing TOML to `<plan>`.
- MUST set an exit status code as described in the [Buildpack Interface](#buildpack-interface) section.

For each `/bin/detect`, the Build Plan received on `stdin` MUST be a map derived from the combined Build Plan contributions of all previous `/bin/detect` executables.
In order to make contributions to the Build Plan, a `/bin/detect` executable MUST write entries to `<plan>` as top-level, TOML-formatted objects.

The lifecycle MUST construct this map such that the top-level values from later buildpacks override the entire top-level values from earlier buildpacks.
The lifecycle MUST NOT include any changes in this map that are contributed by optional buildpacks that returned non-zero exit statuses.
The final Build Plan is the fully-merged map that includes the contributions of the final `/bin/detect` executable.

The lifecycle MAY execute each `/bin/detect` within a group in parallel.
Therefore, reading from `stdin` in `/bin/detect` MUST block until the previous `/bin/detect` finishes executing.

The lifecycle MUST run `/bin/detect` for all buildpacks in a group in a container using common stack with a common set of mixins.
The lifecycle MUST fail detection if any of those buildpacks does not list that stack in `buildpack.toml`.
The lifecycle MUST fail detection if any of those buildpacks specifies a mixin associated with that stack in `buildpack.toml` that is unavailable in the container.

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
- The final ordered group of buildpacks determined during detection,

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

1. Read the Build Plan to determine what dependencies to provide.
2. Provide the application with dependencies for launch in `<layers>/<layer>`.
3. Provide subsequent buildpacks with dependencies in `<layers>/<layer>`.
4. Compile the application source code into object code.
5. Remove application source code that is not necessary for launch.
6. Provide start command in `<layers>/launch.toml`.

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
- The final ordered group of buildpacks determined during detection,
- A directory containing application source code,
- A Build Plan processed by previous `/bin/detect` and `/bin/build` executions,
- Any `<layers>/<layer>.toml` files placed on the filesystem during the analysis phase,
- Any locally cached `<layers>/<layer>` directories, and
- Bash version 3 or greater,

For each buildpack in the group in order, the lifecycle MUST execute `/bin/build`.

1. **IF** the exit status of `/bin/build` is non-zero, \
   **THEN** the lifecycle MUST fail the build.

2. **IF** the exit status of `/bin/build` is zero,
   1. **IF** there are additional buildpacks in the group, \
      **THEN** the lifecycle MUST proceed to the next buildpack's `/bin/build`.

   2. **IF** there are no additional buildpacks in the group, \
      **THEN** the lifecycle MUST proceed to the export phase.

For each `/bin/build` executable in each buildpack, the lifecycle:

- MUST provide a Build Plan to `stdin` of `/bin/build` that is the final Build Plan from the detection phase without any top-level entries that were claimed by previous `/bin/build` executables during the build phase.
- MUST configure the build environment as defined in the [Environment](#environment) section.
- MUST provide path arguments to `/bin/build` as defined in the [Buildpack Interface](#buildpack-interface) section.

Correspondingly, each `/bin/build` executable:

- MAY read or write to the `<app>` directory.
- MAY read the build environment as defined in the [Environment](#environment) section.
- MAY read a Build Plan from `stdin`.
- MAY claim entries in the Build Plan so that they are not received by subsequent `/bin/build` executables during the build phase.
- MAY log output from the build process to `stdout`.
- MAY emit error, warning, or debug messages to `stderr`.
- MAY write a list of possible commands for launch to `<layers>/launch.toml`.
- MAY write a list of sub-paths within `<app>` to `<layers>/launch.toml`.
- MAY write values that should persist to subsequent builds in `<layers>/store.toml`.
- MAY modify or delete any existing `<layers>/<layer>` directories.
- MAY modify or delete any existing `<layers>/<layer>.toml` files.
- MAY create new `<layers>/<layer>` directories.
- MAY create new `<layers>/<layer>.toml` files.
- MAY name any new `<layers>/<layer>` directories without restrictions except those imposed by the filesystem.
- SHOULD NOT use the `<app>` directory to store provided dependencies.

#### Build Plan Entry Claims

A buildpack MAY claim entries in the Build Plan by writing claims to `<plan>` that correspond to entries in the original Build Plan generated during the detection phase.
When an entry is claimed, the lifecycle MUST remove the entry from the Build Plan that is provided via `stdin` to subsequent `/bin/build` executables.

A buildpack MAY write replacement TOML metadata in the entry contents that refines the original contents of the Build Plan entry with information that could not be determined during the detection phase.
The lifecycle MUST NOT make this replacement TOML metadata accessible to subsequent buildpacks.

A buildpack MAY write a Build Plan claim entry that does not correspond to an entry in the original Build Plan.
The lifecycle MUST NOT make this replacement TOML metadata accessible to subsequent buildpacks.

When the build is complete, a BOM (bill-of-materials) MAY be generated for auditing purposes.
If generated, this BOM MUST contain
- All entries from the original Build Plan generated during the detection phase,
- All non-empty entry claims created by `/bin/build` executables such that they override corresponding the Build Plan entries from the detection phase, and
- All entry claims that do not correspond to original Build Plan entries.

#### Layers

A buildpack MAY create, modify, or delete `<layers>/<layer>/` directories and `<layers>/<layer>.toml` files as specified in the [Layer Types](#layer-types) section.

To decide what layer operations are appropriate, the buildpack should consider:

- Whether files in the `<app>` directory have changed since the layer was created.
- Whether the environment has changed since the layer was created.
- Whether the buildpack version has changed since the layer was created.
- Whether new application dependency versions have been made available since the layer was created.

Additionally, a buildpack MAY specify sub-paths within `<app>` as `slices` in `launch.toml`.
Separate layers MUST be created during the export phase for each slice with one or more files or directories.
This minimizes data transfer when the app directory contains a known set of files.

## Phase #4: Export

![Export](img/export.svg)

### Purpose

The purpose of export is to create a new OCI image using a combination of remote layers, local `<layers>/<layer>` layers, and the processed `<app>` directory.

### Process

**GIVEN:**
- The `<layers>` directories provided to each buildpack during the build phase,
- The `<app>` directory processed by the buildpacks during the build phase,
- The buildpack IDs associated with the buildpacks used during the build phase, in order of execution,
- A reference to the most recent version of the run image associated with the stack and mixins,
- A reference to the old OCI image processed during the analysis phase, if available, and
- A tag for a new OCI image,

**IF** the run image, old OCI image, and new OCI image are not all present in the same image store, \
**THEN** the lifecycle SHOULD fail the export process or inform the user that export performance is degraded.

For each `<layers>/<layer>.toml` file that specifies `launch = true`,

1. **IF** a corresponding `<layers>/<layer>` directory is present locally, \
   **THEN** the lifecycle MUST
   1. Convert this directory to a layer.
   2. Transfer the layer to the same image store as the old OCI image.
   3. Ensure the absolute path of `<layers>/<layer>` is preserved in the transferred layer.
   4. Collect a reference to the transferred layer.
2. **IF** a corresponding `<layers>/<layer>` directory is not present locally, \
   **THEN** the lifecycle MUST
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
- An optional process type specified by `CNB_PROCESS_TYPE`, and
- Bash version 3 or greater,

When the OCI image is launched,

1. The lifecycle MUST source each file in each `<layers>/<layer>/profile.d` directory using the same Bash shell process,
   1. Firstly, in order of `/bin/build` execution used to construct the OCI image.
   2. Secondly, in alphabetically ascending order by layer directory name.
   3. Thirdly, in alphabetically ascending order by file name.

2. In the Bash shell process used to source the `profile.d` scripts, the lifecycle MUST source `<app>/.profile` if it is present.

3. If `CMD` in the container configuration is not empty, the lifecycle MUST join each argument with a space and execute the resulting command in the container using the Bash shell process used to source the `profile.d` scripts.

4. If `CMD` in the container configuration is empty,
   1. **IF** the `CNB_PROCESS_TYPE` environment variable is set,
      1. **IF** the value of `CNB_PROCESS_TYPE` corresponds to a process in `<layers>/launch.toml`, \
         **THEN** the lifecycle MUST execute the corresponding command in the container using the Bash shell process used to source the `profile.d` scripts.

      2. **IF** the value of `CNB_PROCESS_TYPE` does not correspond to a process in `<layers>/launch.toml`, \
         **THEN** launch fails.

   2. **IF** the `CNB_PROCESS_TYPE` environment variable is not set,
      1. **IF** there is a process with a `web` process type in `<layers>/launch.toml`, \
         **THEN** the lifecycle MUST execute the corresponding command in the container using the Bash shell process used to source the `profile.d` scripts.

      2. **IF** there is not a process with a `web` process type in `<layers>/launch.toml`, \
         **THEN** launch fails.

When executing a process with Bash, the lifecycle SHOULD replace the Bash process in memory with the resulting command process if possible.

## Development Setup

### Purpose

The purpose of development setup is to create a containerized environment for developing or testing application source code.

During the development setup phase, typical buildpacks might:

1. Read the Build Plan to determine what dependencies to provide.
2. Provide dependencies in `<layers>/<layer>` for development commands and for subsequent buildpacks.
3. Provide a command to start a development server in `<layers>/launch.toml`.
4. Provide a command to run a test suite in `<layers>/launch.toml`.

### Process

**GIVEN:**
- The final ordered group of buildpacks determined during detection,
- A directory containing application source code,
- A Build Plan processed by previous `/bin/detect` and `/bin/develop` executions,
- The most recent local cached `<layers>/<layer>/` directories from a development setup of a version of the application source code, and
- Bash version 3 or greater,

For each buildpack in the group in order, the lifecycle MUST execute `/bin/develop`.

1. **IF** the exit status of `/bin/develop` is non-zero, \
   **THEN** the lifecycle MUST fail the development setup.

2. **IF** the exit status of `/bin/develop` is zero,
   1. **IF** there are additional buildpacks in the group, \
      **THEN** the lifecycle MUST proceed to the next buildpack's `/bin/develop`.

   2. **IF** there are no additional buildpacks in the group, \
      **THEN** the lifecycle MUST launch the process type specified by `CNB_PROCESS_TYPE`.

For each `/bin/develop` executable in each buildpack, the lifecycle:

- MUST provide a Build Plan to `stdin` of `/bin/develop` that is the final Build Plan from the detection phase without any top-level entries that were claimed by previous `/bin/develop` executables during the build phase.
- MUST configure the build environment as defined in the [Environment](#environment) section.
- MUST provide path arguments to `/bin/develop` as defined in the [Buildpack Interface](#buildpack-interface) section.

Correspondingly, each `/bin/develop` executable:

- MAY read from the app directory.
- MAY write files to the app directory in an idempotent manner.
- MAY read the build environment as defined in the [Environment](#environment) section.
- MAY read a Build Plan from `stdin`.
- MAY claim entries in the Build Plan so that they are not received by subsequent `/bin/develop` executables during the development setup.
- MAY log output from the build process to `stdout`.
- MAY emit error, warning, or debug messages to `stderr`.
- MAY write a list of possible commands for launch to `<layers>/launch.toml`.
- MAY write values that should persist to subsequent builds in `<layers>/store.toml`.
- MAY modify or delete any existing `<layers>/<layer>` directories.
- MAY modify or delete any existing `<layers>/<layer>.toml` files.
- MAY create new `<layers>/<layer>` directories.
- MAY create new `<layers>/<layer>.toml` files.
- MAY name any new `<layers>/<layer>` directories without restrictions except those imposed by the filesystem.
- SHOULD NOT use the `<app>` directory to store provided dependencies.
- SHOULD NOT specify any slices within `launch.toml`, as they are only used to generate OCI image layers.

#### Build Plan Entry Claims

In order to claim entries in the Build Plan, a buildpack MUST write an entry claim file `<plan>/<name>` such that `<name>` matches the name of the desired Build Plan entry.
When an entry is claimed, the lifecycle MUST remove the entry from the build plan that is provided via `stdin` to subsequent `/bin/develop` executables.

A buildpack MAY write replacement TOML metadata to an entry claim file that refines the original contents of the build plan entry with information that could not be determined during the detection phase.
However, the lifecycle MUST NOT make this replacement TOML metadata accessible to subsequent buildpacks.

When the build is complete, a BOM (bill-of-materials) MAY be generated for auditing purposes.
If generated, this BOM MUST contain
- All entries from the original build plan generated during the detection phase and
- All non-empty entry claim files created by `/bin/develop` executables
such that the non-empty entry claims override the original build plan entries from the detection phase.

#### Layers

Layers designated `cache = true` in `<layers>/<layer>.toml` MAY be persisted to the next development setup.
Layers not designated `cache = true` in `<layers>/<layer>.toml` MUST be deleted before the next development setup.
Layers designated `launch = true` in `<layers>/<layer>.toml` MUST be made accessible to the development commands as defined in the [Environment](#environment) section.
Layers designated `build = true` in `<layers>/<layer>.toml` MUST be made accessible to subsequent buildpacks as defined in the [Environment](#environment) section.

A buildpack MAY create, modify, or delete `<layers>/<layer>/` directories and `<layers>/<layer>.toml` files.

To decide what layer operations are appropriate, the buildpack should consider:

- Whether files in the `<app>` directory have changed since the layer was created.
- Whether the environment has changed since the layer was created.
- Whether the buildpack version has changed since the layer was created.
- Whether new application dependency versions have been made available since the layer was created.

#### Execution

After the last `/bin/develop` finishes executing,

1. **IF** the `CNB_PROCESS_TYPE` environment variable is set,
   1. **IF** the value of `CNB_PROCESS_TYPE` corresponds to a process in `<layers>/launch.toml`, \
      **THEN** the lifecycle MUST execute the corresponding command in the container using Bash.

   2. **IF** the value of `CNB_PROCESS_TYPE` does not correspond to a process in `<layers>/launch.toml`, \
      **THEN** the lifecycle MUST fail development setup.

2. **IF** the `CNB_PROCESS_TYPE` environment variable is not set,
   1. **IF** there is a process with a `web` process type in `<layers>/launch.toml`, \
      **THEN** the lifecycle MUST execute the corresponding command in the container using Bash.

   2. **IF** there is not a process with a `web` process type in `<layers>/launch.toml`, \
      **THEN** the lifecycle MUST fail development setup.

When executing a process with Bash, the lifecycle SHOULD replace the Bash process in memory with the resulting command process if possible.

## Environment

### Provided by the Lifecycle

The following environment variables MUST be set by the lifecycle during the build and launch phases in order to make buildpack dependencies accessible.

During the build phase, each variable designated for build MUST contain absolute paths of all previous buildpacks’ `<layers>/<layer>/` directories that are designated for build.

When the exported OCI image is launched, each variable designated for launch MUST contain absolute paths of all buildpacks’ `<layers>/<layer>/` directories that are designated for launch.

In either case,

- The lifecycle MUST order all `<layer>` paths to reflect the reversed order of the buildpack group.
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

| Env Variable    | Description                          | Detect | Build | Launch
|-----------------|--------------------------------------|--------|-------|--------
| `CNB_STACK_ID`  | Chosen stack ID                      | [x]    | [x]   |
| `BP_*`          | User-provided variable for buildpack | [x]    | [x]   |
| `BPL_*`         | User-provided variable for profile.d |        |       | [x]
| `HOME`          | Current user's home directory        | [x]    | [x]   | [x]

During the detection and build phases, the lifecycle MUST provide any user-provided environment variables as files in `<platform>/env/` with file names and contents matching the environment variable names and contents.

The lifecycle MUST NOT set user-provided environment variables in the environment of `/bin/detect` or `/bin/build` directly.

Buildpacks MAY use the value of `CNB_STACK_ID` to modify their behavior when executed on different stacks.

The environment variable prefix `CNB_` is reserved.
It MUST NOT be used for environment variables that are not defined in this specification or approved extensions.

### Provided by the Buildpacks

During the build phase, buildpacks MAY write environment variable files to `<layers>/<layer>/env/`,  `<layers>/<layer>/env.build/`, and `<layers>/<layer>/env.launch/` directories.

For each `<layers>/<layer>/` designated as a build layer, for each file written to `<layers>/<layer>/env/` or `<layers>/<layer>/env.build/` by `/bin/build`, the lifecycle MUST modify an environment variable in subsequent executions of `/bin/build`.

For each file written to `<layers>/<layer>/env.launch/` by `/bin/build`, the lifecycle MUST modify an environment variable when the OCI image is launched.

The lifecycle MUST set the name of the environment variable to the name of the file up to the first period (`.`) or to the end of the name if no periods are present.

If the environment variable file name has no period-delimited suffix, then the value of the environment variable MUST be a concatenation of the file contents and the contents of other identically named files delimited by the OS path list separator.
Within that environment variable value,
- Later buildpacks' environment variable file contents MUST precede earlier buildpacks' environment variable file contents.
- Environment variable file contents originating from the same buildpack MUST be sorted alphabetically descending by associated layer name.
- Environment variable file contents originating in the same layer MUST be sorted such that file contents in `<layers>/<layer>/env.build/` or `<layers>/<layer>/env.launch/` precede file contents in `<layers>/<layer>/env/`.

If the environment variable file name ends in `.append`, then the value of the environment variable MUST be a concatenation of the file contents and the contents of other identically named files without any delimitation.
Within that environment variable value,
- Later buildpacks' environment variable file contents MUST precede earlier buildpacks' environment variable file contents.
- Environment variable file contents originating from the same buildpack MUST be sorted alphabetically descending by associated layer name.
- Environment variable file contents originating in the same layer MUST be sorted such that file contents in `<layers>/<layer>/env.build/` or `<layers>/<layer>/env.launch/` precede file contents in `<layers>/<layer>/env/`.

If the environment variable file name ends in `.override`, then the value of the environment variable MUST be the file contents or the contents of another identically named file.
For that environment variable value
- Later buildpacks' environment variable file contents MUST override earlier buildpacks' environment variable file contents.
- For environment variable file contents originating from the same buildpack, file contents that are later (when sorted alphabetically ascending by associated layer name) MUST override file contents that are earlier.
- Environment variable file contents originating in the same layer MUST be sorted such that file contents in `<layers>/<layer>/env.build/` or `<layers>/<layer>/env.launch/` override file contents in `<layers>/<layer>/env/`.

In all cases, file contents MUST NOT be evaluated by a shell or otherwise modified before inclusion in environment variable values.

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
mixins = ["<mixin name>"]
build-images = ["<build image tag>"]
run-images = ["<run image tag>"]

[metadata]
# buildpack-specific data
```

Buildpack authors MUST choose a globally unique ID, for example: "io.buildpacks.ruby".

The buildpack ID:
- MUST only contain numbers, letters, and the characters `.`, `/`, and `-`.
- MUST NOT be `config` or `app`.
- MUST NOT be identical to any other buildpack ID when using a case-insensitive comparison.

The buildpack version:
- MUST NOT be `latest`.

Stack authors MUST choose a globally unique ID, for example: "io.buildpacks.mystack".

The stack ID:
- MUST only contain numbers, letters, and the characters `.`, `/`, and `-`.
- MUST NOT be identical to any other stack ID when using a case-insensitive comparison.

The stack `build-images` and `run-images` are suggested sources of images for platforms that are unaware of the stack ID. Buildpack authors MUST ensure that these images include all mixins specified in `mixins`.

### launch.toml (TOML)

```toml
[[processes]]
type = "<process type>"
command = "<command>"

[[slices]]
paths = ["<app sub-path glob>"]
```

The buildpack MAY specify any number of processes or slices.

For each process, the buildpack MUST specify:

- A unique process type for each entry within a `launch.toml` file.
- A command that is valid when executed using the Bash 3+ shell.

For each slice, buildpacks MUST specify zero or more path globs such that each path is either:

- Relative to the root of the app directory without traversing outside of the app directory.
- Absolute and contained within the app directory.

Path globs MUST:

- Follow the pattern syntax defined in the [Go standard library](https://golang.org/pkg/path/filepath/#Match).
- Match zero or more files or directories.

The lifecycle MUST process each slice as if all files matched in preceding slices no longer exists in the app directory.

The lifecycle MUST accept slices that do not contain any files or directory. However, the lifecycle MAY warn about such slices.

The lifecycle MUST include all unmatched files in the app directory in any number of additional layers in the OCI image.

### store.toml (TOML)

```toml
[metadata]
# buildpack-specific data
```

### Build Plan (TOML)

```toml
[<dependency name>]
version = "<dependency version>"

[<dependency name>.metadata]
# buildpack-specific data
```

For a given dependency, the buildpack MAY specify:

- The dependency version. If a version range is needed, semver notation SHOULD be used to specify the range.
- The ID of the buildpack that will provide the dependency.

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
