# Buildpack API v3 - Buildpack Interface Specification

This document specifies the interface between a single lifecycle and one or more buildpacks.

A lifecycle is a program that uses buildpacks to transform application source code into an OCI image containing the compiled application.

This is accomplished in four steps:
1. **Detection,** where an optimal selection of compatible buildpacks is chosen.
2. **Analysis,** where metadata about OCI layers generated during a previous build are made available to buildpacks.
3. **Build,** where buildpacks use that metadata to generate only the OCI layers that need to be replaced.
4. **Export,** where the remote layers are replaced by the generated layers.

Additionally, a lifecycle MAY use buildpacks to create a containerized environment for developing or testing application source code.

This is accomplished in two steps:
1. **Detection,** where an optimal selection of compatible buildpacks is chosen.
2. **Development,** where the lifecycle uses those buildpacks to create a containerized development environment. 

## Buildpack Interface

### Key

 Mark | Meaning                                   
------|-------------------------------------------
 A    | Single copy provided for all buildpacks
 E    | Different copy provided for each buildpack
 I    | Image repository for storage
 C    | Cache for storage
 R    | Read-only
 *    | Buildpack-specific content
 #    | Platform-specific content

The following specifies the interface implemented by executables in each buildpack.
The lifecycle MUST invoke these executables as described in the Process section.

### Detection

Executable: `/bin/detect`, Working Dir: `<app[AR]>`

 Input         | Description
---------------|----------------------------------------------
 `/dev/stdin`  | Merged plan from previous detections (TOML)

 Output        | Description
---------------|----------------------------------------------
 [exit status] | Pass (0), fail (100), or error (1-99, 101+)
 `/dev/stdout` | Updated plan for subsequent detections (TOML)
 `/dev/stderr` | Detection logs (all)

###  Build

Executable: `/bin/build <platform[AR]> <cache[EC]> <launch[EI]>`, Working Dir: `<app[AI]>`

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

### Development

Executable: `/bin/develop <platform[A]> <cache[EC]>`, Working Dir: `<app[A]>`

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

The purpose of detection is to find an ordered group of buildpacks to use during the build step.
These buildpacks must be compatible with the app.

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
         **THEN** the group MUST be provided to the analysis and build steps with buildpacks that did not return exit status zero removed.
         
The `/bin/detect` executable in each buildpack, when executed:

1. MAY examine the app directory.
2. MAY receive a TOML-formatted map called a Build Plan on `stdin`.
3. MAY output changes to the Build Plan on `stdout`.
 
For each `/bin/detect`, the Build Plan received on `stdin` MUST be a combined map derived from the output of all previous `/bin/detect` executables.
The lifecycle MUST construct this map such that the top-level values from later buildpacks override the entire top-level values from earlier buildpacks. 
The lifecycle MUST NOT include any changes in this map that are output by optional buildpacks that returned non-zero exit statuses. 
The final Build Plan is the complete combined map that includes the output of the final `/bin/detect` executable.

The lifecycle MAY execute each `/bin/detect` within a group in parallel.
Therefore, reading from `stdin` in `/bin/detect` blocks until the previous `/bin/detect` in the group closes `stdout`.

The lifecycle MUST run `/bin/detect` for all buildpacks in a group on a common stack.
The lifecycle MUST fail detection if any of those buildpacks does not list that stack in `buildpack.toml`.

### Analysis

The lifecycle SHOULD attempt to locate a reference to an OCI image from a previous build that:
- Was created using some version of the same application source code.
- Is readable by the lifecycle.
- Was created using the lifecycle.
- Is as recent as possible.

The lifecycle MAY skip analysis if no such image can be located.

**GIVEN:**
- A reference to the previously created OCI image described above and
- The final ordered group of buildpacks determined during detection,

For each buildpack in the group,

1. Any `<launch>/<layer>.toml` files that were present at the end of the previous OCI image build are retrieved.
2. Those `<layer>.toml` files are placed on the filesystem so that they appear in the buildpack's `<launch>/` directory during the build step.

The lifecycle MUST NOT download any filesystem layers from the previous OCI image.

After analysis, the lifecycle proceeds to the build step.

### Build

**GIVEN:**
- The final ordered group of buildpacks determined during detection and
- A directory containing application source code and
- The final Build Plan and
- Any `<launch>/<layer>.toml` files placed on the filesystem during the analysis step and
- The most recent local `<cache>` directories from a build of a version of the application source code,

For each buildpack in the group in order, the lifecycle MUST execute `/bin/build`.

1. **IF** the exit status of `/bin/build` is non-zero, \
   **THEN** the lifecycle MUST fail the build.
   
2. **IF** the exit status of `/bin/build` is zero,
   1. **IF** there are additional buildpacks in the group, \
      **THEN** the lifecycle MUST proceed to the next buildpack's `/bin/build`.
      
   2. **IF** there are no additional buildpacks in the group, \
      **THEN** the lifecycle MUST proceed to the export step.

For each `/bin/build` executable in each buildpack, the lifecycle:

- MUST provide a Build Plan to `stdin` of `/bin/build`.
- MUST configure the build environment as defined in the Environment section.
- MUST provide path arguments to `/bin/build` as defined in the Buildpack Interface section.
- MAY provide an empty `<cache>` directory if the platform does not make it available.  

Correspondingly, each `/bin/build` executable:

- MAY read or write to the `<app>` directory.
- MAY read a Build Plan from `stdin`.
- MAY supply dependencies in `<cache>/<layer>` directories.
- MAY supply dependencies in new `<launch>/<layer>` directories and create corresponding `<launch>/<layer>.toml` files.
- MAY name any new `<layer>` directories without restrictions except those imposed by the filesystem.
- SHOULD NOT use the `<app>` directory to store provided dependencies.
- MUST NOT provide paths for `<launch>/<layer>` directories to other buildpack.

The buildpack SHOULD use the contents of a given pre-existing `<launch>/<layer>.toml` file to decide:

1. Whether to create a `<launch>/<layer>` directory that will replace a remote layer during the export step.
2. Whether to remove the `<launch>/<layer>.toml` file to delete a remote layer during the export step.
3. Whether to modify the `<launch>/<layer>.toml` file for the next build.

To make this decision, the buildpack should consider:

1. Whether files in the `<app>` directory are sufficiently similar to the previous build.
2. Whether the environment is sufficiently similar to the previous build.
3. Whether files in `<cache>` directories are sufficiently similar to the previous build.
4. Whether the buildpack version has changed since the previous build.

### Export

The lifecycle MUST NOT download any filesystem layers from the previous OCI image.

### Launch

### Development Setup

**GIVEN:**
- The final ordered group of buildpacks determined during detection and
- A directory containing application source code and
- The final Build Plan and
- The most recent local `<cache>` directories from a development setup of a version of the application source code,

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
- MUST configure the build environment as defined in the Environment section.
- MUST provide path arguments to `/bin/develop` as defined in the Buildpack Interface section.
- MAY provide an empty `<cache>` directory if the platform does not make it available.  

Correspondingly, each `/bin/develop` executable:

- MAY read from the app directory.
- MAY write files to the app directory in an idempotent manner.
- MAY read a Build Plan from `stdin`.
- MAY supply dependencies in `<cache>/<layer>` directories.
- MAY name any new `<layer>` directories without restrictions except those imposed by the filesystem.
- SHOULD NOT use the `<app>` directory to store provided dependencies.

## Environment

The following environment variables MUST be set by the lifecycle in order to make buildpack dependencies accessible.

During the build process, each variable MUST contain absolute paths of all previous buildpacks’ `<cache>/<layer>` directories.

When the output image is launched, each variable MUST contain absolute paths of all buildpacks’ `<launch>/<layer>` directories.

In either case,

- The lifecycle MUST order all `<layer>` paths to reflect the order of the buildpack group.
- The lifecycle MAY order all `<layer>` paths provided by a given buildpack alphabetically.
- The lifecycle MUST separate each path with the OS path separator (e.g., `:` on Linux).
- Buildpacks MUST NOT depend on the order of layer paths provided by a given buildpack.

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

During the build process, buildpacks MAY write files to `<cache>/<layer>/env/` directories.
These files MUST contain filesystem paths delimited by the operating-system-specific path separator.

For each file written to `<cache>/<layer>/env/` by `/bin/build`, the lifecycle MUST modify an environment variable in subsequent executions of `/bin/build` such that

- The name of an environment variable MUST be identical to the name of the file.
- The value of the environment variable MUST be a concatenation of the file contents and the contents of other identically named files in other `<cache>/<layer>/env/` directories delimited by the operating-system-specific path separator.
- Within the environment variable value, earlier buildpacks' file contents MUST proceed later buildpacks' file contents.
- Within the environment variable value, file contents originating from the same buildpack MAY be sorted alphabetically by associated layer name.

The lifecycle MUST place any user-provided environment variables into files in `<platform>/env/` with file names and contents matching the environment variable names and contents.
The lifecycle MUST NOT set user-provided environment variables in the environment of `/bin/build` directly.

## Artifact Format

### Buildpack Package

Buildpacks SHOULD be stored as gzip-compressed tarballs with extension `.tgz`.

Buildpacks MUST contain `/buildpack.toml` and `/bin/detect`.

Buildpacks MUST contain one or both of `/bin/build` and `/bin/develop`.

## Data Format

### buildpack.toml (TOML)

```toml
[buildpack]
id = "<buildpack ID, globally unique, ex. io.buildpacks.ruby, not 'app'>"
name = "<buildpack name>"
version = "<buildpack version, semver compliant>"

[[stacks]]
id = "<stack ID, globally unique, ex. io.buildpacks.stack>"

[metadata]
<buildpack-specific data>
```

### launch.toml (TOML)

```toml
[[processes]]
type = "<process type>"
command = "<command>"
```

### develop.toml (TOML)

```toml
[[processes]]
type = "<process type>"
command = "<command>"
```

### Build Plan (TOML)

```toml
[<dependency name>]
version = "<dependency version or *>"

[<dependency name>.metadata]
<buildpack-specific data>
```

## Layer Caching

The purpose of layer caching is to
1. Minimize the execution time of the build process.
2. Minimize persistent disk usage. 

This is achieved by
1. Reducing the number of build operations.
2. Reducing data transfer. 
3. Enabling de-duplication of stored image layers.


If the buildpack creates a <launch>/<layer>.toml file, the contents of that file are stored as a LABEL on the resulting image and recovered on the next build at the same location.

If <launch>/<layer>.toml is modified, the changes are persisted to the image metadata.

If a <launch>/<layer>.toml file exists and a <launch>/<layer>/ directory exists, the contents of <launch>/<layer>/ will replace the remote layer.

If a <launch>/<layer>.toml file exists and no <launch>/<layer>/ directory exists, then the contents of <launch>/<layer>/ are recovered from the previous build.

If no <launch>/<layer>.toml exists after the build, no corresponding remote layer is included in the updated image.
