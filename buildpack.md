# Buildpack API v3 - Interface Specification

This document specifies the interface between a lifecycle and a buildpack. 

## Buildpack Executable Interface

 Key | Meaning                                   
-----|-------------------------------------------
 A   | Single copy provided for all buildpacks
 E   | Different copy provided for each buildpack
 I   | Image repository for storage
 C   | Cache for storage
 R   | Read-only
 *   | Buildpack-specific content
 #   | Platform-specific content

The following specifies the interface provided by executables in each buildpack.
The executables MUST be invoked by the lifecycle as described in the Process section.

Executable: `/bin/detect`, Working Dir: <app[AR]>

 Input         | Description
---------------|----------------------------------------------
 `/dev/stdin`  | Merged plan from previous detections (TOML)

 Output        | Description
---------------|----------------------------------------------
 [exit status] | Pass (0), fail (100), or error (1-99, 101+)
 `/dev/stdout` | Updated plan for subsequent detections (TOML)
 `/dev/stderr` | Detection logs (all)

Executable: `/bin/build <platform[AR]> <cache[EC]> <launch[EI]>`, Working Dir: <app[AI]>

 Input                         | Description
-------------------------------|----------------------------------------------
 `/dev/stdin`                  | Build plan from detection (TOML)
 `<platform>/env/`             | User-provided environment variables for build
 `<platform>/#`                | Platform-specific extensions

 Output                        | Description
-------------------------------|----------------------------------------------
 [exit status]                 | Success (0) or failure (1+)
 `/dev/stdout`                 | Logs (info)
 `/dev/stderr`                 | Logs (warnings, errors)
 `<cache>/<layer>/bin/`        | Binaries for subsequent buildpacks
 `<cache>/<layer>/lib/`        | Libraries for subsequent buildpacks
 `<cache>/<layer>/include/`    | C/C++ headers for subsequent buildpacks
 `<cache>/<layer>/pkgconfig/`  | Search path for pkg-config
 `<cache>/<layer>/env/`        | Appended env vars for subsequent buildpacks
 `<cache>/<layer>/*`           | Other cached content
 `<launch>/launch.toml`        | Launch metadata (see File: launch.toml)
 `<launch>/<layer>.toml`       | Layer content metadata (see Layer Caching)
 `<launch>/<layer>/bin/`       | Binaries for launch
 `<launch>/<layer>/lib/`       | Shared libraries for launch
 `<launch>/<layer>/profile.d/` | Scripts sourced by bash before launch
 `<launch>/<layer>/*`          | Other content for launch

Executable: `/bin/develop <platform[A]> <cache[EC]>`, Working Dir: <app[A]>

 Input                        | Description
------------------------------|----------------------------------------------
 [exit status]			      | Success (0) or failure (1+)
 `/dev/stdin`                 | Build plan from detection (TOML)
 `/dev/stdout`                | Logs (info)
 `/dev/stderr`                | Logs (warnings, errors)
 `<platform>/env/`            | User-provided environment variables for build
 `<platform>/#`               | Platform-specific extensions
 `<cache>/develop.toml`       | Development metadata (see File: develop.toml)
 `<cache>/<layer>/bin/`       | Binaries for subsequent buildpacks
 `<cache>/<layer>/lib/`       | Libraries for subsequent buildpacks
 `<cache>/<layer>/include/`   | C/C++ headers for subsequent buildpacks
 `<cache>/<layer>/pkgconfig/` | Search path for pkg-config
 `<cache>/<layer>/env/`       | Append env vars for subsequent buildpacks
 `<cache>/<layer>/*`          | Other cached content

## App Interface

 Output           | Description
------------------|----------------------------------------------
 `<app>/.profile` | Script sourced by bash before launch

## Process

### Detection

The purpose of detection is to find a suitable group of buildpacks to use during the build phase.

A group MUST be selected by the lifecycle as follows.

Given an ordered list of buildpack groups, each containing an ordered list of buildpacks, for each group,

1. For each buildpack in order, `/bin/detect` MUST be executed. \
   **IF** exit status of `/bin/detect` is 0 \
   **OR** the buildpack is marked optional \
   **THEN** the lifecycle MUST proceed to the next buildpack in the group, \
   **OTHERWISE** detection of this group MUST fail, and detection MUST proceed to the next group if present.

2. After all buildpack `/bin/detect` executables have finished executing,
   **IF** all non-optional buildpacks return exit status 0 \
   **AND** at least one buildpack has exited with status 0 \
   **THEN** the chosen group MUST be the ordered set of buildpacks that returned exit status 0, \
   **OTHERWISE** detection of this group MUST fail, and detection MUST proceed to the next group if present.

The `/bin/detect` executable in each buildpack, when executed:

1. MAY examine the app directory
2. MAY receive a Build Plan (TOML) from stdin
3. MAY output changes (TOML) for the Build Plan to stdout

Changes to the Build Plan MUST be new top-level keys that MUST replace the entire value of existing keys, if present.

Each `/bin/detect` within a group MAY be run in parallel.
Therefore, reading from stdin MAY block until the previous buildpack in the group closes stdout.

The output of the last buildpack to stdout is the final Build Plan. 

The lifecycle MUST run `/bin/detect` from all buildpacks in all groups on a common stack.
All buildpacks in all groups MUST assume they will be executed on the stack, which they MAY query via the `PACK_STACK_NAME` environment variable.

## Build

Given the final group of buildpacks from detection and a build plan,

1. Cache layers SHOULD be restored if present and available in order to optimize performance.

2. **IF** the user or platform is able to identify an image with similar layers, \
   **SUCH AS** an image generated from a previous build of the same app, \
   **THEN** Layer Metadata MUST be read from that image and written to disk as defined in the Layer Caching section.
   
3. For each buildpack in order, `/bin/build` MUST be executed.
   **IF** exit status of `/bin/build` is 0
   **THEN** the lifecycle MUST proceed to the next build
   **OTHERWISE** the build MUST fail.

4. New layers and Layer Metadata are stored as defined in the Layer Caching section.

The `/bin/build` executable in each buildpack, when executed:

1. MAY read or write to the app directory.
2. MAY read a Build Plan (TOML) from stdin.
3. MAY supply dependencies in <layer> directories in <launch> or <cache>.
4. MAY name each <layer> directory without restrictions.
5. SHOULD NOT use the app directory to store provided dependencies.

Correspondingly, the lifecycle:

1. MUST provide a Build Plan (TOML) to stdin of `/bin/build`.
2. MUST configure the build environment as defined in the Environment section.
3. MUST provide path arguments to `/bin/build` as defined in the Executable Interface section.

## Development

Given the final group of buildpacks from detection and a build plan,

1. Cache layers SHOULD be restored if present and available in order to optimize performance.

2. For each buildpack in order, `/bin/develop` MUST be executed.
   **IF** exit status of `/bin/develop` is 0
   **THEN** the lifecycle MUST proceed to the next build
   **OTHERWISE** the development setup MUST fail.

The `/bin/develop` executable in each buildpack, when executed:

1. MAY read from the app directory.
2. MAY write files to the app directory in an idempotent manner.
3. MAY read a Build Plan (TOML) from stdin
4. MAY supply dependencies in <layer> directories in <cache>.
5. MAY name each <layer> directory without restrictions.
6. SHOULD NOT use the app directory to store provided dependencies.

Correspondingly, the lifecycle:

1. MUST provide a Build Plan (TOML) to stdin of `/bin/develop`.
2. MUST configure the build environment as defined in the Environment section.
3. MUST provide path arguments `/bin/develop` as defined in the Executable Interface section.

## Artifact Format

### Buildpack Package

Buildpacks SHOULD be stored as gzip-compressed tarballs with extension `.tgz`.

Buildpacks MUST contain a `/buildpack.toml` file.

Buildpacks MAY contain any of `/bin/detect`, `/bin/build`, or `/bin/develop`.

## Data Format

### buildpack.toml (TOML)

```toml
[buildpack]
id = "<buildpack ID, globally unique, ex. io.buildpacks.ruby, not 'app'>"
name = "<buildpack name>"
version = "<buildpack version, semver compliant>"

[[stacks]]
id = "<stack ID, globally unique, ex. io.buildpacks.stack>"
run-contrib = "/path/to/run.Dockerfile"
build-contrib = "/path/to/build.Dockerfile" 

[metadata]
<buildpack-specific data>
```

### launch.toml (TOML)

```toml
processes = [{type = "<process type>", command = "<command>"}]
```

### develop.toml (TOML)

```toml
processes = [{type = "<process type>", command = "<command>"}]
```

### Build Plan (TOML)

```toml
[<dependency name>]
version = "<dependency version or *>"
metadata = {<buildpack-specific data>}
```

### Build Metadata (TOML)
```toml
processes = [{type = "<process type>", command = "<command>"}]
buildpacks = ["<buildpack ID: string>"]
```

## Environment

The following environment variables MUST be set by the lifecycle in order to make buildpack dependencies accessible.

During the build process, each variable MUST contain absolute paths of all previous buildpacks’ <cache>/<layer> directories.

When the output image is launched, each variable MUST contain absolute paths of all buildpacks’ <launch>/<layer> directories.

In either case,

- The lifecycle MUST order all layer paths to reflect the order of the buildpack group.
- The lifecycle MAY order layer paths provided by a particular given buildpack alphabetically.
- The lifecycle MUST separate each path with the OS path separator (`:` on Linux).
- Buildpacks MUST NOT depend on the order of layer paths provided by a particular given buildpack.

 Env Variable      | Layer Path   | Contents         | Build | Launch
-------------------|--------------|------------------|-------|--------
 `PATH`            | `/bin`       | binaries         | [x]   | [x]
 `LD_LIBRARY_PATH` | `/lib`       | shared libraries | [x]   | [x]
 `LIBRARY_PATH`    | `/lib`       | static libraries | [x]   |
 `CPATH`           | `/include`   | header files     | [x]   |
 `PKG_CONFIG_PATH` | `/pkgconfig` | pc files         | [x]   |

The following additional environment variables MAY be read by the buildpacks and MUST NOT be overridden by the lifecycle.

 Env Variable    | Description                            | Detect | Build | Launch
-----------------|----------------------------------------|--------|-------|--------
 `PACK_SERVICES` | OSBAPI service bindings                |        |       | [x]
 `PACK_SERVICES` | OSBAPI service bindings (restricted)   | [x]    | [x]   |
 `PACK_STACK_ID` | Chosen stack ID                        | [x]    | [x]   |
 `BP_*`          | User-specified variable for buildpack  | [x]    | [x]   |
 `BPL_*`		 | User-specified variable for profile.d  |        |       | [x]

During the build process, buildpacks MAY write files to `<cache>/<layer>/env/`. 
For subsequent buildpack builds, the lifecycle MUST set an environment variable for each file such that,

- The name of the environment variable is identical to the file name.
- The value of the environment variable is identical to the file contents.
- 

Environment variables supplied by buildpacks in <cache>/<layer>/env/ directories are set automatically for all subsequent buildpacks by the platform. They are not set during launch.

Build environment variables supplied by the user are made available in <platform>/env/ but not set automatically by the platform.

User-provided environment variables intended for build and launch SHOULD NOT come from the same list.
The user SHOULD be encouraged to define them separately.
Platform operators MAY determine the initial build-time and runtime environment.

The lifecycle MUST NOT assume that all platforms provide an identical environment.

## Layer Caching

Layer caching results in fast builds when only some application bits are changed.

This is achieved by minimizing both build operations and data transfer.

If the buildpack creates a <launch>/<layer>.toml file, the contents of that file are stored as a LABEL on the resulting image and recovered on the next build at the same location.

If <launch>/<layer>.toml is modified, the changes are persisted to the image metadata.

If a <launch>/<layer>.toml file exists and a <launch>/<layer>/ directory exists, the contents of <launch>/<layer>/ will replace the remote layer.

If a <launch>/<layer>.toml file exists and no <launch>/<layer>/ directory exists, then the contents of <launch>/<layer>/ are recovered from the previous build.

If no <launch>/<layer>.toml exists after the build, no corresponding remote layer is included in the updated image.

When deciding whether to keep the remote layer, the buildpack SHOULD use the contents of <launch>/<layer>.toml to decide:

1. Whether files in the app are sufficiently similar to the previous build

2. Whether build-time environment variables are sufficiently similar to the previous build

3. Whether files in the <cache> directories are sufficiently similar to the previous build

4. Whether the buildpack version has changed since the previous build

Note:

* The <launch> directory SHOULD never be made accessible to subsequent buildpacks.

* File system layers SHOULD never be downloaded from the previous sluglet.

