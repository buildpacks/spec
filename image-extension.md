# Image Extension Interface Specification

This document specifies the interface between a lifecycle program and one or more image extensions.

The lifecycle program uses image extensions to generate Dockerfiles that can be used to extend build and/or run base images prior to buildpacks builds.

## Table of Contents

<!-- Using https://github.com/yzhang-gh/vscode-markdown to manage toc -->

## Image Extension API Version

This document accompanies Buildpack API version `0.8`.

## Image Extension Interface

Unless otherwise noted, image extensions are expected to conform to the [Buildpack Interface Specification](buildpack.md).

### Detection

Executable: `/bin/detect`, Working Dir: `<app[AR]>`

Image extensions participate in the buildpack [detection](buildpack.md#detection) process, with the same interface for `/bin/detect`. However:
- Detection is optional for image extensions, and they are assumed to pass detection when `/bin/detect` is not present.
- Image extensions MUST only output `provides` entries to the build plan. They MUST NOT output `requires`.

### Build-Ext

Executable: `/bin/build`, Working Dir: `<app[AR]>`

Image extensions participate in a build-ext process that is similar to the buildpack [build](buildpack.md#build) process, with the same interface for `/bin/build`. However:
- Image extensions' `/bin/build` MUST NOT write to the app directory.
- `$CNB_LAYERS_DIR` MUST be the absolute path of an `<output>` directory and MUST NOT be the path of the buildpack layers directory.
- If an image extension is missing `/bin/build`, the image extension root MUST be treated as a pre-populated `<output>` directory.

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
sbom-formats = [ "<string>" ]

[[extension.licenses]]
type = "<string>"
uri = "<uri>"
```

Image extension authors MUST choose a globally unique ID, for example: "io.buildpacks.apt".

The image extension `id`, `version`, `api`, `licenses`, and `sbom-formats` entries MUST follow the requirements defined in the [Buildpack Interface Specification](buildpack.md).

### launch.toml (TOML) for extensions; for buildpacks see the [Buildpack Interface Specification](buildpack.md)

```toml
[[args]]
name = "<build arg name>"
value = "<build arg value>"

[[labels]]
key = "<label key>"
value = "<label valu>"
```

The image extension MAY specify any number of args or labels.

For each arg, the image extension:
- MUST specify a `name` to be the name of a build argument that will be provided to all output Dockerfiles or run.Dockerfiles when extending the run base image.
- MUST specify a `value` to be the value of the build argument when it is provided.

The image extension `labels` entries MUST follow the requirements defined in the [Buildpack Interface Specification](buildpack.md).

### build.toml (TOML) for buildpacks; for image extensions see the [Image Extension Specification](image-extension.md)

```toml
[[args]]
name = "<build arg name>"
value = "<build arg value>"

[[unmet]]
name = "<dependency name>"
```

For each arg, the image extension:

- MUST specify a `name` to be the name of a build argument that will be provided to all output Dockerfiles or build.Dockerfiles when extending the build base image.
- MUST specify a `value` to be the value of the build argument when it is provided.

The image extension `unmet` entries MUST follow the requirements defined in the [Buildpack Interface Specification](buildpack.md).
