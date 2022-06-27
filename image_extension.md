# Image Extension Interface Specification (**experimental**)

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

This document accompanies Buildpack API version `0.9`.

## Image Extension Interface

Unless otherwise noted, image extensions are expected to conform to the [Buildpack Interface Specification](buildpack.md).

### Detection

Executable: `/bin/detect`, Working Dir: `<app[AR]>`

Image extensions participate in the buildpack [detection](buildpack.md#detection) process, with the same interface for `/bin/detect`. However:
- Detection is optional for image extensions, and they are assumed to pass detection when `/bin/detect` is not present.
- If an image extension is missing `/bin/detect`, the image extension root MUST be treated as a pre-populated `<output>` directory.
- Image extensions MUST only output `provides` entries to the build plan. They MUST NOT output `requires`.

### Generation

Executable: `/bin/generate`, Working Dir: `<app[AR]>`

Image extensions participate in a generation process that is similar to the buildpack [build](buildpack.md#build) process, with an interface that is similar to `/bin/build`. However:
- Image extensions' `/bin/generate` MUST NOT write to the app directory.
- Instead of the `CNB_LAYERS_DIR` input, image extensions MUST receive a `CNB_OUTPUT_DIR` which MUST be the absolute path of an `<output>` directory and MUST NOT be the path of the buildpack layers directory.
- If an image extension is missing `/bin/generate`, the image extension root MUST be treated as a pre-populated `<output>` directory.

## Phase: Generation

### Purpose

The purpose of the generation phase is to generate Dockerfiles that can be used to define the runtime base image.

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
- MAY write a run.Dockerfile to the `<output>` directory. This file MUST adhere to the requirements listed below.
- MUST NOT write SBOM (Software-Bill-of-Materials) files as described in the [Software-Bill-of-Materials](#software-bill-of-materials) section.

#### Dockerfile Requirements

run.Dockerfiles:

- MAY contain a single `FROM` instruction
- MUST NOT contain any other instructions

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
```

Image extension authors MUST choose a globally unique ID, for example: "io.buildpacks.apt".

The image extension `id`, `version`, `api`, `licenses`, and `sbom-formats` entries MUST follow the requirements defined in the [Buildpack Interface Specification](buildpack.md).

### build.toml (TOML)

This section describes the `build.toml` output by image extensions; for buildpacks see the [Buildpack Interface Specification](buildpack.md).

```toml
[[unmet]]
name = "<dependency name>"
```

The image extension `unmet` entries MUST follow the requirements defined in the [Buildpack Interface Specification](buildpack.md).
