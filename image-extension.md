# Image Extension Interface Specification

This document specifies the interface of image extensions.

Image extensions are dynamically-generated or static build-time and runtime modules that mutate the pre-build base image.

## Table of Contents

<!-- Using https://github.com/yzhang-gh/vscode-markdown to manage toc -->


## Image Extension API Version

This document specifies Image Extension API version `0.1`.

Image Extension API versions:
 - MUST be in form `<major>.<minor>` or `<major>`, where `<major>` is equivalent to `<major>.0`
 - When `<major>` is greater than `0` increments to `<minor>` SHALL exclusively indicate additive changes

## Image Extension Interface

An image extension is used to apply a `Dockerfile` at the platform's discretion.

The image extensions MAY contain a `/bin/build` and `/bin/detect` executable.
Each extension MUST have an `extension.toml` file in its root directory.

Extensions participate in the buildpack [detection](#detector) process, with the same `UID`, `GID`, and interface for `/bin/detect`. However:
- Detection is optional for extensions, and they are assumed to pass detection when `/bin/detect` is not present. A `/bin/detect` that exits with a 0 exit code passes detection, and fails otherwise.
- Extensions MUST only output `provides` entries to the build plan. They MUST NOT output `requires`.
- Extensions MUST all precede buildpacks in [order](#ordertoml-toml) definitions.
- Extensions MUST always be optional.

Extensions MUST generate `Dockerfile`s before the buildpack [build](#builder) phase. To generate Dockerfiles, the lifecycle MUST execute the extension's `/bin/build` executable with the same `UID`, `GID`, and interface as buildpacks. However:

- Extension's `/bin/build` MUST NOT write to the app directory.
- Extension's `/bin/build` MAY be executed in parallel.
- The buildpack `<layers>` directory MUST be replaced by an `<output>` directory.
- If an extension is missing `/bin/build`, the extension root MUST be treated as a pre-populated `<output>` directory.

After `/bin/build` executes, the `<output>` directory MAY contain
- `build.toml`, with the same contents as a normal buildpack's `build.toml`, but
  - With an additional `args` table array with `name` and `value` fields that are provided as build args to `build.Dockerfile` or `Dockerfile`
- `launch.toml`, with the same contents as a normal buildpack's `launch.toml`, but
  - Without the `processes` table array
  - Without the `slices` table array
  - With an additional `args` table array with `name` and `value` fields that are provided as build args to `run.Dockerfile` or `Dockerfile`
- Either `Dockerfile` or either or both of `build.Dockerfile` and `run.Dockerfile`. The `build.Dockerfile`, `run.Dockerfile`, and `Dockerfile` target the builder image, runtime base image, or both base images, respectively.

If no Dockerfiles are present, `/bin/build` may still consume build plan entries and add metadata to `build.toml`/`launch.toml`.

`Dockerfile`s MUST be applied to their corresponding base images after all extensions are executed and before any regular buildpacks are executed. `Dockerfile`s MUST be applied in the order determined during buildpack detection.

All `Dockerfile`s MUST be provided with `base_image` and `build_id` args.
The `base_image` arg allows the Dockerfile to reference the original base image.
The `build_id` arg allows the Dockerfile to invalidate the cache after a certain layer and must be defaulted to `0`. The executor of the Dockerfile will provide the `build_id` as a UUID (this eliminates the need to track this variable).
When the `$build_id` arg is referenced in a `RUN` instruction, all subsequent layers will be rebuilt on the next build (as the value will change).

Build args specified in `build.toml` MUST be provided to `build.Dockerfile` or `Dockerfile` (when applied to the build-time base image).
Build args specified in `launch.toml` MUST be provided to `run.Dockerfile` or `Dockerfile` (when applied to the runtime base image).

A runtime base image extension MAY indicate that it preserves ABI compatibility by adding the label `io.buildpacks.rebasable=true`. The `image-extender` MUST set `io.buildpacks.rebasable=true` on the final image only if `io.buildpacks.rebasable=true` on the base image and for all image extensions.

### Image Extensions Directory Layout

The image extensions directory MUST contain unarchived extensions such that:

- Each top-level directory is an image extension ID.
- Each second-level directory is an image extension version.

## Image Extension Examples

### Example: App-specified Dockerfile Extension

This example extension would allow an app to provide runtime and build-time base image extensions as "run.Dockerfile" and "build.Dockerfile."
The app developer can decide whether the extensions are rebasable.

##### `/cnb/ext/com.example.appext/bin/build`
```
#!/bin/sh
[ -f build.Dockerfile ] && cp build.Dockerfile "$1/"
[ -f run.Dockerfile ] && cp run.Dockerfile "$1/"
```

### Example: RPM Dockerfile Extension (app-based)

This example extension would allow a builder to install RPMs for each language runtime, based on the app directory.

Note: The Dockerfiles referenced must disable rebasing, and build times will be slower compared to buildpack-provided runtimes.

##### `/cnb/ext/com.example.rpmext/bin/build`
```
#!/bin/sh
[ -f Gemfile.lock ] && cp "$CNB_BUILDPACK_DIR/Dockerfile-ruby" "$1/Dockerfile"
[ -f package.json ] && cp "$CNB_BUILDPACK_DIR/Dockerfile-node" "$1/Dockerfile"
```


### Dockerfiles for Base Images

The same Dockerfile format may be used to create new base images or modify existing base images outside of the app build process (e.g., before creating a builder). Any specified labels override existing values.

Dockerfiles that are used to create a base image must create a `/cnb/image/genpkgs` executable that outputs a [CycloneDX](https://cyclonedx.org)-formatted list of packages in the image with PURL IDs when invoked. This executable is executed after all Dockerfiles are applied, and the output replaces the label `io.buildpacks.sbom`. This label doubles as a Software Bill-of-Materials for the base image. In the future, this label will serve as a starting point for the application SBoM.

### Example Dockerfiles

Dockerfile used to create a runtime base image:

```
ARG base_image
FROM ${base_image}
ARG build_id=0

LABEL io.buildpacks.image.distro=ubuntu
LABEL io.buildpacks.image.version=18.04
LABEL io.buildpacks.rebasable=true

ENV CNB_USER_ID=1234
ENV CNB_GROUP_ID=1235

RUN groupadd cnb --gid ${CNB_GROUP_ID} && \
  useradd --uid ${CNB_USER_ID} --gid ${CNB_GROUP_ID} -m -s /bin/bash cnb

USER ${CNB_USER_ID}:${CNB_GROUP_ID}

COPY genpkgs /cnb/image/genpkgs
```

`run.Dockerfile` for use with the example `app.run.Dockerfile.out` extension that always installs the latest version of curl:
```
ARG base_image
FROM ${base_image}
ARG build_id=0

RUN echo ${build_id}

RUN apt-get update && apt-get install -y curl && rm -rf /var/lib/apt/lists/*
```
(note: this Dockerfile disables rebasing, as OS package installation is not rebasable)

`run.Dockerfile` for use with the example `app.run.Dockerfile.out` extension that installs a special package to /opt:
```
ARG base_image
FROM ${base_image}
ARG build_id=0

LABEL io.buildpacks.rebasable=true

RUN curl -L https://example.com/mypkg-install | sh # installs to /opt
```
(note: rebasing is explicitly allowed because only a single directory in /opt is created)


## Data Format

### Files

#### `extension.toml` (TOML)

```toml
api = "<image extension API version>"

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

**The image extension ID:**

*Key: `id = "<extension ID>"`*
- MUST only contain numbers, letters, and the characters `.`, `/`, and `-`.
- MUST NOT be `config` or `app`.
- MUST NOT be identical to any other buildpack ID when using a case-insensitive comparison.

The image extension version:
- MUST be in the form `<X>.<Y>.<Z>` where `X`, `Y`, and `Z` are non-negative integers and must not contain leading zeros.
   - Each element MUST increase numerically.
   - Buildpack authors will define what changes will increment `X`, `Y`, and `Z`.

**The image extension API:**

*Key: `api = "<image extension API version>"`*
 - MUST be in form `<major>.<minor>` or `<major>`, where `<major>` is equivalent to `<major>.0`
 - MUST describe the implemented platform API.
 - SHOULD indicate the lowest compatible `<minor>` if extension behavior is consistent with multiple `<minor>` versions of a given `<major>`

**The image extension licenses:**

The `[[extension.licenses]]` table is optional and MAY contain a list of image extension licenses where:

- `type` - This MAY use the SPDX 2.1 license expression, but is not limited to identifiers in the SPDX Licenses List.
- `uri` - If this buildpack is using a nonstandard license, then this key MAY be specified in lieu of or in addition to `type` to point to the license.
