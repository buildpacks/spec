# Image Extension Interface Specification

This document specifies the interface between a lifecycle program and one or more image extensions.

## Table of Contents

<!-- Using https://github.com/yzhang-gh/vscode-markdown to manage toc -->
- [Image Extension Interface Specification](#image-extension-interface-specification)
  - [Table of Contents](#table-of-contents)
  - [Image Extension API Version](#image-extension-api-version)
  - [Image Extension Interface](#image-extension-interface)
    - [Detection](#detection)
    - [Generation](#generation)
  - [Phase: Generation](#phase-generation)
    - [Purpose](#purpose)
    - [Process](#process)
      - [Dockerfile Requirements](#dockerfile-requirements)
  - [Data Format](#data-format)
    - [Files](#files)
    - [extension.toml (TOML)](#extensiontoml-toml)

## Image Extension API Version

This document accompanies Buildpack API version `0.10`.

## Image Extension Interface

Unless otherwise noted, image extensions are expected to conform to the [Buildpack Interface Specification](buildpack.md).

### Detection

Executable: `/bin/detect`, Working Dir: `<app[AR]>`

Image extensions participate in the buildpack [detection](buildpack.md#detection) process, with the same interface for `/bin/detect`. However:
- Detection is optional for image extensions, and they are assumed to pass detection when `/bin/detect` is not present.
- If an image extension is missing `/bin/detect`, the image extension root `/detect` directory MUST be treated as a pre-populated `<output>` directory.
- Instead of the `CNB_BUILDPACK_DIR` input, image extensions MUST receive a `CNB_EXTENSION_DIR` which MUST be the absolute path of the extension root directory.
- Image extensions MUST only output `provides` entries to the build plan. They MUST NOT output `requires`.

### Generation

Executable: `/bin/generate`, Working Dir: `<app[AR]>`

Image extensions participate in a generation process that is similar to the buildpack [build](buildpack.md#build) process, with an interface that is similar to `/bin/build`. However:
- Image extensions' `/bin/generate` MUST NOT write to the app directory.
- Instead of the `CNB_LAYERS_DIR` input, image extensions MUST receive a `CNB_OUTPUT_DIR` which MUST be the absolute path of an `<output>` directory and MUST NOT be the path of the buildpack layers directory.
- Instead of the `CNB_BUILDPACK_DIR` input, image extensions MUST receive a `CNB_EXTENSION_DIR` which MUST be the absolute path of the extension root directory.
- If an image extension is missing `/bin/generate`, the image extension root `/generate` directory MUST be treated as a pre-populated `<output>` directory.

## Phase: Generation

### Purpose

The purpose of the generation phase is to generate Dockerfiles that can be used to define the build and/or runtime base image. The generation phase MUST NOT be run for Windows builds.

### Process

**GIVEN:**
- The final ordered group of image extensions determined during the detection phase,
- A directory containing application source code,
- The Buildpack Plan,
- An `<output>` directory used to store generated artifacts,
- A shell, if needed,

For each image extension in the group in order, the lifecycle MUST execute `/bin/generate`.

1. **If** the exit status of `/bin/generate` is non-zero, \
   **Then** the lifecycle MUST fail the build.

2. **If** the exit status of `/bin/generate` is zero,
    1. **If** there are additional image extensions in the group, \
       **Then** the lifecycle MUST proceed to the next image extension's `/bin/generate`.

    2. **If** there are no additional image extensions in the group, \
       **Then** the lifecycle MUST proceed to the build phase.

For each `/bin/generate` executable in each image extension, the lifecycle:

- MUST provide path arguments to `/bin/generate` as described in the [generation](#generation) section.
- MUST configure the build environment as described in the [Environment](buildpack.md#environment) section.
- MUST provide all `<plan>` entries that were required by any buildpack in the group during the detection phase with names matching the names that the image extension provided.

Correspondingly, each `/bin/generate` executable:

- MAY read from the `<app>` directory.
- MUST NOT write to the `<app>` directory.
- MAY read the build environment as described in the [Environment](buildpack.md#environment) section.
- MAY read the Buildpack Plan.
- MAY log output from the build process to `stdout`.
- MAY emit error, warning, or debug messages to `stderr`.
- MAY write either or both of `build.Dockerfile` and `run.Dockerfile` to the `<output>` directory. This file MUST adhere to the requirements listed below.
- MAY create the following folders in the `<output>` directory with an arbitrary content:
  
  either:

  - `context`

  or the image-specific folders:

  - `context.run`
  - `context.build`
- MAY write key-value pairs to `<output>/extend-config.toml` that are provided as build args to build.Dockerfile when extending the build image.
- MAY write telemetry data to `$CNB_OTEL_LOG_PATH` in [OpenTelemetry File Exporter format](https://opentelemetry.io/docs/specs/otel/protocol/file-exporter/) when provided.
- MUST NOT write SBOM (Software-Bill-of-Materials) files as described in the [Software-Bill-of-Materials](#software-bill-of-materials) section.

#### Context Folders

- The `<output>/context` folder MUST NOT be created together with any combination of the image-specific folders.
- If the folder `<output>/context` is present it will be set as the build context during the `extend` phase of the build and run images.
- If the folder `<output>/context.run` is present it will be set as the build context during the `extend` phase of the run image only.
- If the folder `<output>/context.build` is present it will be set as the build context during the `extend` phase of the build image only.
- If none of these folders is not present, the build context defaults to the `<app>` folder.

#### Dockerfile Requirements

A `run.Dockerfile`

- MAY contain a single `FROM` instruction
- MUST NOT contain any other `FROM` instructions
- MAY contain `ADD`, `ARG`, `COPY`, `ENV`, `LABEL`, `RUN`, `SHELL`, `USER`, and `WORKDIR` instructions
- SHOULD NOT contain any other instructions
- SHOULD use the `build_id` build arg to invalidate the cache after a certain layer. When the `$build_id` build arg is referenced in a `RUN` instruction, all subsequent layers will be rebuilt on the next build (as the value will change); the `build_id` build arg SHOULD be defaulted to 0 if used (this ensures portability)
- SHOULD NOT edit `<app>`, `<layers>`, or `<platform>` directories (see the [Platform Interface Specification](platform.md)) as changes will not be persisted
- SHOULD use the `user_id` and `group_id` build args to reset the image config's `User` field to its original value if any `USER` instructions are employed
- SHOULD set the label `io.buildpacks.rebasable` to `true` to indicate that any new run image layers are safe to rebase on top of new runtime base images
  - For the final image to be rebasable, all applied Dockerfiles must set this label to `true`

A `build.Dockerfile`

- MUST begin with:
```bash
ARG base_image
FROM ${base_image}
```
- MUST NOT contain any other `FROM` instructions
- MAY contain `ADD`, `ARG`, `COPY`, `ENV`, `LABEL`, `RUN`, `SHELL`, `USER`, and `WORKDIR` instructions
- SHOULD NOT contain any other instructions
- SHOULD use the `build_id` build arg to invalidate the cache after a certain layer. When the `$build_id` build arg is referenced in a `RUN` instruction, all subsequent layers will be rebuilt on the next build (as the value will change); the `build_id` build arg SHOULD be defaulted to 0 if used (this ensures portability)
- SHOULD NOT edit `<app>`, `<layers>`, or `<platform>` directories (see the [Platform Interface Specification](platform.md)) as changes will not be persisted
- SHOULD use the `user_id` and `group_id` build args to reset the image config's `User` field to its original value if any `USER` instructions are employed

## Phase: Extension

### Purpose

The purpose of the extension phase is to apply the Dockerfiles generated in the generation phase to the appropriate base image. The extension phase MUST NOT be run for Windows builds.

### Process

**GIVEN:**
- The final ordered group of Dockerfiles generated during the generation phase,
- A list of build args for each Dockerfile specified during the generation phase,

For each Dockerfile in the group in order, the lifecycle MUST apply the Dockerfile to the base image as follows:

- The lifecycle MUST provide each Dockerfile with:
- A `base_image` build arg
    - For the first Dockerfile, the value MUST be the original base image.
    - When there are multiple Dockerfiles, the value MUST be the intermediate image generated from the application of the previous Dockerfile.
- A `build_id` build arg
    - The value MUST be a UUID
- `user_id` and `group_id` build args
    - For the first Dockerfile, the values MUST be the original `uid` and `gid` from the `User` field of the config for the original base image.
    - When there are multiple Dockerfiles, the values MUST be the `uid` and `gid` from the `User` field of the config for the intermediate image generated from the application of the previous Dockerfile.

## Data Format

### Files

### extension.toml (TOML)

This section describes the 'Extension descriptor'.

```toml
api = "<buildpack API version>"

[extension]
id = "<extension ID>"
name = "<extension name>"
version = "<extension version>"
homepage = "<extension homepage>"
description = "<extension description>"
keywords = [ "<string>" ]

[[extension.licenses]]
type = "<string>"
uri = "<uri>"

[[targets]]
os = "<OS name>"
arch = "<architecture>"
variant = "<architecture variant>"
[[targets.distros]]
name = "<OS distribution name>"
version = "<OS distribution version>"

[metadata]
# extension-specific data
```

Image extension authors MUST choose a globally unique ID, for example: "io.buildpacks.apt".

The image extension `id`, `version`, `api`, and `licenses` entries MUST follow the requirements defined in the [Buildpack Interface Specification](buildpack.md).

An extension descriptor MAY specify `targets` following the requirements defined in the [Buildpack Interface Specification](buildpack.md).

### extend-config.toml (TOML)

```toml
[[build.args]]
name = "<build arg name>"
value = "<build arg value>"

[[run.args]]
name = "<build arg name>"
value = "<build arg value>"
```

The image extension MAY specify any number of args.

For each `[[build.args]]`, the image extension:
- MUST specify a `name` to be the name of a build argument that will be provided to any output `build.Dockerfile` when extending the build base image.
- MUST specify a `value` to be the value of the build argument that is provided.

For each `[[run.args]]`, the image extension:
- MUST specify a `name` to be the name of a build argument that will be provided to any output `run.Dockerfile` when extending the runtime base image.
- MUST specify a `value` to be the value of the build argument that is provided.

### Build Plan (TOML)

See the [Buildpack Interface Specification](buildpack.md).

### Buildpack Plan (TOML)

See the [Buildpack Interface Specification](buildpack.md). Image extensions MUST satisfy all entries in the Buildpack Plan.
