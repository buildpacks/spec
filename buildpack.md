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
11. [Data Format](#data-format)
    1. [launch.toml (TOML)](#launchtoml-toml)
    2. [Build Plan (TOML)](#build-plan-toml)
    3. [Buildpack Plan (TOML)](#buildpack-plan-toml)
    4. [Bill-of-Materials (TOML)](#bill-of-materials-toml)
    5. [Layer Content Metadata (TOML)](#layer-content-metadata-toml)
    6. [buildpack.toml (TOML)](#buildpacktoml-toml)

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
| `<platform>/env/` | User-provided environment variables for build
| `<platform>/#`    | Platform-specific extensions

| Output             | Description
|--------------------|----------------------------------------------
| [exit status]      | Pass (0), fail (100), or error (1-99, 101+)
| `/dev/stdout`      | Logs (info)
| `/dev/stderr`      | Logs (warnings, errors)
| `<plan>`           | Contributions to the the Build Plan (TOML)


###  Build

Executable: `/bin/build <layers[EIC]> <platform[AR]> <plan[E]>`, Working Dir: `<app[AI]>`

| Input             | Description
|-------------------|----------------------------------------------
| `$0`              | Absolute path of `/bin/build` executable
| `<plan>`          | Relevant [Buildpack Plan entries](#buildpack-plan-toml) from detection (TOML)
| `<platform>/env/` | User-provided environment variables for build
| `<platform>/#`    | Platform-specific extensions

| Output                         | Description
|--------------------------------|-----------------------------------------------
| [exit status]                  | Success (0) or failure (1+)
| `/dev/stdout`                  | Logs (info)
| `/dev/stderr`                  | Logs (warnings, errors)
| `<plan>`                       | Refinements to the [Buildpack Plan](#buildpack-plan-toml) (TOML)
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
| `<plan>`          | Relevant [Buildpack Plan entries](#buildpack-plan-toml) from detection (TOML)
| `<platform>/env/` | User-provided environment variables for build
| `<platform>/#`    | Platform-specific extensions

| Output                         | Description
|--------------------------------|----------------------------------------------
| [exit status]                  | Success (0) or failure (1+)
| `/dev/stdout`                  | Logs (info)
| `/dev/stderr`                  | Logs (warnings, errors)
| `<plan>`                       | Refinements to the [Buildpack Plan](#buildpack-plan-toml) (TOML)
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

The lifecycle MUST make all launch layers accessible to the app as described in the [Environment](#environment) section.

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

The lifecycle MUST make all build layers accessible to subsequent buildpacks as described in the [Environment](#environment) section.

Before the next re-build:
- If the layer is marked `cache = true`, the lifecycle MAY restore the `<layers>/<layer>/` directory and Layer Content Metadata from any previous build to the same path.
- If the layer is marked `cache = false`, the lifecycle MUST NOT restore the `<layers>/<layer>/` directory or the Layer Content Metadata from any previous build.

#### Other Layers

For layers marked `launch = true` and `build = true`, the most strict requirements of each type apply.

Therefore, the lifecycle MUST consider such layers to be launch layers that are also accessible to subsequent buildpacks as described in the [Environment](#environment) section.

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
- An ordered list of buildpack groups resolved into buildpack implementations as described in [Order Resolution](#order-resolution) and
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
- MAY read the detect environment as described in the [Environment](#environment) section.
- MAY emit error, warning, or debug messages to `stderr`.
- MAY augment the Build Plan by writing TOML to `<plan>`.
- MUST set an exit status code as described in the [Buildpack Interface](#buildpack-interface) section.

In order to make contributions to the Build Plan, a `/bin/detect` executable MUST write entries to `<plan>` in two sections: `requires` and `provides`.
Additionally, these two sections MAY be repeated together inside of an `or` array at the top-level.
Each `requires` and `provides` section MUST be a list of entries formatted as described in the [Build Plan](#build-plan-toml) format section.

Each pairing of `requires` and `provides` sections (at the top level, or inside of an `or` array) is a potential Build Plan.

For a given buildpack group, a sequence of trials is generated by selecting a single potential Build Plan from each buildpack in a left-to-right, depth-first order.
The group fails to detect if all trials fail to detect.

For each trial,
- If a required buildpack provides a dependency that is not required by the same buildpack or a subsequent buildpack, the trial MUST fail to detect.
- If a required buildpack requires a dependency that is not provided by the same buildpack or a previous buildpack, the trial MUST fail to detect.
- If an optional buildpack provides a dependency that is not required by the same buildpack or a subsequent buildpack, the optional buildpack MUST be excluded from the build phase and its requires and provides MUST be excluded from the Build Plan.
- If an optional buildpack requires a dependency that is not provided by the same buildpack or a previous buildpack, the optional buildpack MUST be excluded from the build phase and its requires and provides MUST be excluded from the Build Plan.
- Multiple buildpacks MAY require or provide the same dependency.

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
7. Refine the Buildpack Plan in `<plan>` with more exact metadata.

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
- Bash version 3 or greater, if needed,

For each buildpack in the group in order, the lifecycle MUST execute `/bin/build`.

1. **IF** the exit status of `/bin/build` is non-zero, \
   **THEN** the lifecycle MUST fail the build.

2. **IF** the exit status of `/bin/build` is zero,
   1. **IF** there are additional buildpacks in the group, \
      **THEN** the lifecycle MUST proceed to the next buildpack's `/bin/build`.

   2. **IF** there are no additional buildpacks in the group, \
      **THEN** the lifecycle MUST proceed to the export phase.

For each `/bin/build` executable in each buildpack, the lifecycle:

- MUST provide path arguments to `/bin/build` as described in the [Buildpack Interface](#buildpack-interface) section.
- MUST configure the build environment as described in the [Environment](#environment) section.
- MUST provide all `<plan>` entries that were required by any buildpack in the group during the detection phase with names matching the names that the buildpack provided.

Correspondingly, each `/bin/build` executable:

- MAY read or write to the `<app>` directory.
- MAY read the build environment as described in the [Environment](#environment) section.
- MAY read the Buildpack Plan.
- MAY augment the Buildpack Plan with more refined metadata.
- MAY remove entries with duplicate names in the Buildpack Plan to refine the metadata.
- MAY remove all entries of the same name from the Buildpack Plan to defer those entries to subsequent `/bin/build` executables.
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

#### Buildpack Plan Entry Refinements

A buildpack MAY refine entries in `<plan>` by replacing any entries of the same name with a single entry of that name.
The single entry MAY include additional metadata that could not be determined during the detection phase.

A buildpack MAY add extra entries to `<plan>` that do not correspond to entries added during the detection phase.

The lifecycle MUST NOT allow any entries with names matching those in `<plan>` at the end of `/bin/build` to be available in subsequent buildpacks' `<plan>`s.

The lifecycle MUST defer any entries whose names were entirely removed from `<plan>` to the next buildpack that provided entries with those names during the detection phase.

When the build is complete, a BOM (Bill-of-Materials) MAY be generated for auditing purposes.
If generated, this BOM MUST contain all entries in each `<plan>` at the end of each `/bin/build` execution.

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
3. The lifecycle MAY add Config labels to the new OCI image, composed of
   - A label with the key `"io.buildpacks.app.source"` with a [value describing the source location of the app](#io.buildpacks.app.source-oci-image-label).

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
- Bash version 3 or greater, if needed,

First, the lifecycle MUST locate a start command and choose an execution strategy.

To locate a start command,

1. **IF** `CMD` in the container configuration is not empty,
    **THEN** the value of `CMD` is chosen as the start command.

2. **IF** `CMD` in the container configuration is empty,
   1. **IF** the `CNB_PROCESS_TYPE` environment variable is set,
      1. **IF** the value of `CNB_PROCESS_TYPE` corresponds to a process in `<layers>/launch.toml`, \
         **THEN** the lifecycle MUST choose the corresponding process as the start command.

      2. **IF** the value of `CNB_PROCESS_TYPE` does not correspond to a process in `<layers>/launch.toml`, \
         **THEN** launch fails.

   2. **IF** the `CNB_PROCESS_TYPE` environment variable is not set,
      1. **IF** there is a process with a `web` process type in `<layers>/launch.toml`, \
         **THEN** the lifecycle MUST choose the corresponding process as the start command.

      2. **IF** there is not a process with a `web` process type in `<layers>/launch.toml`, \
         **THEN** launch fails.

To choose an execution strategy,

1. **IF** the value of `CMD` is chosen as the start command,
   1. **IF** the first parameter of `CMD` is not `--`,
      **THEN** the lifecycle MUST invoke the value as a command using Bash with subsequent entries as arguments.

   2. **IF** the first parameter of `CMD` is `--` and the length of `CMD` is greater than one,
      **THEN** the lifecycle MUST invoke the second entry using the `execve` syscall with subsequent entries as arguments.


2. **IF** a buildpack-provided process type is chosen as the start command,
   1. **IF** the process type does not have `direct` set to `true`,
      **THEN** the lifecycle MUST invoke the value of `command` as a command using Bash with values of `args` provided as arguments.

   2. **IF** the process type does have `direct` set to `true`,
      **THEN** the lifecycle MUST invoke the value of `command` using the `execve` syscall with values of `args` provided as arguments.

Given the start command and execution strategy,

1. The lifecycle MUST set all buildpack-provided launch environment variables as described in the [Environment](#environment) section.

2. If using an execution strategy involving Bash, the lifecycle MUST use a single Bash process to
   1. source each file in each `<layers>/<layer>/profile.d` directory,
      1. Firstly, in order of `/bin/build` execution used to construct the OCI image.
      2. Secondly, in alphabetically ascending order by layer directory name.
      3. Thirdly, in alphabetically ascending order by file name.
   2. source `<app>/.profile` if it is present.

3. The lifecycle MUST invoke the start command with the decided execution strategy.

When executing a process using any execution strategy, the lifecycle SHOULD replace the lifecycle process in memory without forking it.
When executing a process with Bash, the lifecycle SHOULD additionally replace the Bash process in memory without forking it.

## Development Setup

### Purpose

The purpose of development setup is to create a containerized environment for developing or testing application source code.

During the development setup phase, typical buildpacks might:

1. Read the Buildpack Plan to determine what dependencies to provide.
2. Provide dependencies in `<layers>/<layer>` for development commands and for subsequent buildpacks.
3. Provide a command to start a development server in `<layers>/launch.toml`.
4. Provide a command to run a test suite in `<layers>/launch.toml`.

### Process

**GIVEN:**
- The final ordered group of buildpacks determined during the detection phase,
- A directory containing application source code,
- The Buildpack Plan,
- The most recent local cached `<layers>/<layer>/` directories from a development setup of a version of the application source code, and
- Bash version 3 or greater,

For each buildpack in the group in order, the lifecycle MUST execute `/bin/develop`.

1. **IF** the exit status of `/bin/develop` is non-zero, \
   **THEN** the lifecycle MUST fail the development setup.

2. **IF** the exit status of `/bin/develop` is zero,
   1. **IF** there are additional buildpacks in the group, \
      **THEN** the lifecycle MUST proceed to the next buildpack's `/bin/develop`.

   2. **IF** there are no additional buildpacks in the group, \
      **THEN** the lifecycle MUST proceed to executing a process type as specified in the [Launch](#launch) section.

For each `/bin/develop` executable in each buildpack, the lifecycle:

- MUST configure the build environment as described in the [Environment](#environment) section.
- MUST provide path arguments to `/bin/develop` as described in the [Buildpack Interface](#buildpack-interface) section.
- MUST provide all `<plan>` entries that were required by any buildpack in the group during the detection phase with names matching the names that the buildpack provided.

Correspondingly, each `/bin/develop` executable:

- MAY read from the app directory.
- MAY write files to the app directory in an idempotent manner.
- MAY read the build environment as described in the [Environment](#environment) section.
- MAY read the Buildpack Plan.
- MAY augment the Buildpack Plan with more refined metadata.
- MAY remove entries with duplicate names in the Buildpack Plan to refine the metadata.
- MAY remove all entries of the same name from the Buildpack Plan to defer those entries to subsequent `/bin/develop` executables.
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

#### Buildpack Plan Entry Refinements

A buildpack MAY refine entries in `<plan>` by replacing any entries of the same name with a single entry of that name.
The single entry MAY include additional metadata that could not be determined during the detection phase.

A buildpack MAY add extra entries to `<plan>` that do not correspond to entries added during the detection phase.

The lifecycle MUST NOT allow any entries with names matching those in `<plan>` at the end of `/bin/develop` to be available in subsequent buildpacks' `<plan>`s.

The lifecycle MUST defer any entries whose names were entirely removed from `<plan>` to the next buildpack that provided entries with those names during the detection phase.

When the build is complete, a BOM (Bill-of-Materials) MAY be generated for auditing purposes.
If generated, this BOM MUST contain all entries in each `<plan>` at the end of each `/bin/develop` execution.

#### Layers

Layers designated `cache = true` in `<layers>/<layer>.toml` MAY be persisted to the next development setup.
Layers not designated `cache = true` in `<layers>/<layer>.toml` MUST be deleted before the next development setup.
Layers designated `launch = true` in `<layers>/<layer>.toml` MUST be made accessible to the development commands as described in the [Environment](#environment) section.
Layers designated `build = true` in `<layers>/<layer>.toml` MUST be made accessible to subsequent buildpacks as described in the [Environment](#environment) section.

A buildpack MAY create, modify, or delete `<layers>/<layer>/` directories and `<layers>/<layer>.toml` files.

To decide what layer operations are appropriate, the buildpack should consider:

- Whether files in the `<app>` directory have changed since the layer was created.
- Whether the environment has changed since the layer was created.
- Whether the buildpack version has changed since the layer was created.
- Whether new application dependency versions have been made available since the layer was created.

## Environment

### Provided by the Lifecycle

#### Layer Paths

The following layer path environment variables MUST be set by the lifecycle during the build and launch phases in order to make buildpack dependencies accessible.

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

##### Delimiter

If the environment variable file name ends in `.delim`, then the file contents MUST be used to delimit any concatenation within the same layer involving that environment variable.
This delimiter MUST override the delimiters below.
If multiple operations apply to the same environment variable, all operations for a given layer containing environment variable files MUST be applied before subsequent layers are considered.

##### Prepend

If the environment variable file name has no period-delimited suffix, then the value of the environment variable MUST be a concatenation of the file contents and the contents of other files representing that environment variable delimited by the OS path list separator.
If the environment variable file name ends in `.prepend`, then the value of the environment variable MUST be a concatenation of the file contents and the contents of other files representing that environment variable.
In either case, within that environment variable value,
- Later buildpacks' environment variable file contents MUST precede earlier buildpacks' environment variable file contents.
- Environment variable file contents originating from the same buildpack MUST be sorted alphabetically descending by associated layer name.
- Environment variable file contents originating in the same layer MUST be sorted such that file contents in `<layers>/<layer>/env.build/` or `<layers>/<layer>/env.launch/` precede file contents in `<layers>/<layer>/env/`.

##### Append

If the environment variable file name ends in `.append`, then the value of the environment variable MUST be a concatenation of the file contents and the contents of other files representing that environment variable.
Within that environment variable value,
- Earlier buildpacks' environment variable file contents MUST precede later buildpacks' environment variable file contents.
- Environment variable file contents originating from the same buildpack MUST be sorted alphabetically ascending by associated layer name.
- Environment variable file contents originating in the same layer MUST be sorted such that file contents in `<layers>/<layer>/env/` precede file contents in `<layers>/<layer>/env.build/` or `<layers>/<layer>/env.launch/`.

##### Override

If the environment variable file name ends in `.override`, then the value of the environment variable MUST be the file contents.
For that environment variable value,
- Later buildpacks' environment variable file contents MUST override earlier buildpacks' environment variable file contents.
- For environment variable file contents originating from the same buildpack, file contents that are later (when sorted alphabetically ascending by associated layer name) MUST override file contents that are earlier.
- Environment variable file contents originating in the same layer MUST be sorted such that file contents in `<layers>/<layer>/env.build/` or `<layers>/<layer>/env.launch/` override file contents in `<layers>/<layer>/env/`.

##### Default

If the environment variable file name ends in `.default`, then the value of the environment variable MUST only be the file contents if the environment variable is empty.
For that environment variable value,
- Earlier buildpacks' environment default variable file contents MUST override later buildpacks' environment variable file contents.
- For default environment variable file contents originating from the same buildpack, file contents that are earlier (when sorted alphabetically ascending by associated layer name) MUST override file contents that are later.
- Default environment variable file contents originating in the same layer MUST be sorted such that file contents in `<layers>/<layer>/env/` override file contents in  `<layers>/<layer>/env.build/` or `<layers>/<layer>/env.launch/`.

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
[[processes]]
type = "<process type>"
command = "<command>"
args = ["<arguments>"]
direct = false

[[slices]]
paths = ["<app sub-path glob>"]
```

The buildpack MAY specify any number of processes or slices.

For each process, the buildpack:

- MUST specify a `type` that is not identical to other process types provided by the same buildpack.
- MUST specify a `command` that is either:
  - A command sequence that is valid when executed using the Bash 3+ shell, if `args` is not specified.
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
version = "<dependency version>"

[requires.metadata]
# buildpack-specific data

[[or]]

[[or.provides]]
name = "<dependency name>"

[[or.requires]]
name = "<dependency name>"
version = "<dependency version>"

[or.requires.metadata]
# buildpack-specific data

```

### Buildpack Plan (TOML)

```toml
[[entries]]
name = "<dependency name>"
version = "<dependency version>"

[entries.metadata]
# buildpack-specific data
```

### Bill-of-Materials (TOML)

```toml
[[entries]]
name = "<dependency name>"
version = "<dependency version>"

[[entries.buildpacks]]
id = "<buildpack ID>"
version = "<buildpack version>"

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


### buildpack.toml (TOML)

```toml
api = "<buildpack api>"

[buildpack]
id = "<buildpack ID>"
name = "<buildpack name>"
version = "<buildpack version>"
clear-env = false

[[order]]
[[order.group]]
id = "<buildpack ID>"
version = "<buildpack version>"
optional = false

[[stacks]]
id = "<stack ID>"
mixins = ["<mixin name>"]

[metadata]
# buildpack-specific data
```

Buildpack authors MUST choose a globally unique ID, for example: "io.buildpacks.ruby".

The buildpack ID:
- MUST only contain numbers, letters, and the characters `.`, `/`, and `-`.
- MUST NOT be `config` or `app`.
- MUST NOT be identical to any other buildpack ID when using a case-insensitive comparison.


If an `order` is specified, then `stacks` MUST NOT be specified.

The buildpack API:
 - MUST be in form `<major>.<minor>` or `<major>`, where `<major>` is equivalent to `<major>.0`
 - MUST describe the implemented buildpack API.
 - SHALL indicate compatibility with a given lifecycle according to the following rules:
    - When `<major>` is `0`, the buildpack is only compatible with lifecycles implementing that exact buildpack API.
    - When `<major>` is greater than `0`, the buildpack is only compatible with lifecycles implementing buildpack API `<major>.<minor>`, where `<major>` of the lifecycle equals `<major>` of the buildpack and `<minor>` of the lifecycle is greater than or equal to `<minor>` of the buildpack.

#### Buildpack Implementations

A buildpack descriptor that specifies `stacks` MUST describe a buildpack that implements the [Buildpack Interface](#buildpack-interface).

Stack authors MUST choose a globally unique ID, for example: "io.buildpacks.mystack".

The stack ID:
- MUST only contain numbers, letters, and the characters `.`, `/`, and `-`.
- MUST NOT be identical to any other stack ID when using a case-insensitive comparison.

#### Order Buildpacks

A buildpack descriptor that specifies `order` MUST be [resolvable](#order-resolution) into an ordering of buildpacks that implement the [Buildpack Interface](#buildpack-interface).

A buildpack reference inside of a `group` MUST contain an `id` and `version`.

### io.buildpacks.app.source OCI Image label
The value of this label:
- MUST be a string of escaped json complying with [RFC 8259](https://tools.ietf.org/html/rfc8259).
- when unescaped, MUST comply with the following schema:
```json
{
  "$schema": "http://json-schema.org/schema#",
  "type": "object",
  "properties": {
    "type": {"type": "string"},
    "version": {},
    "metadata": {}
  }
}
```