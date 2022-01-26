# Distribution Specification

This document specifies the artifact format and the delivery mechanism for the buildpacks core components.

## Table of Contents

<!-- TOC -->

- [Table of Contents](#table-of-contents)
- [Distribution API Version](#distribution-api-version)
- [Artifact Format](#artifact-format)
  - [Buildpackage](#buildpackage)
    - [Labels](#labels)
      - [`io.buildpacks.buildpack.metadata` (JSON)](#iobuildpacksbuildpackmetadata-json)
      - [`io.buildpacks.buildpack.layers` (JSON)](#iobuildpacksbuildpacklayers-json)
  - [Lifecycle](#lifecycle)
    - [Filesystem](#filesystem)
    - [Labels](#labels)
  - [Build Image](#build-image)
    - [Image Configuration](#image-configuration)
    - [Environment Variables](#environment-variables)
    - [Labels](#labels)
  - [Run Image](#run-image)
    - [Image Configuration](#image-configuration)
    - [Environment Variables](#environment-variables)
    - [Labels](#labels)
  - [Builder](#builder)
    - [General Requirements](#general-requirements)
    - [Filesystem](#filesystem)
    - [Environment Variables](#environment-variables)
    - [Labels](#labels)
      - [`io.buildpacks.builder.metadata` (JSON)](#iobuildpacksbuildermetadata-json)

<!-- /TOC -->

## Distribution API Version

This document specifies Distribution API version `0.3`.

Distribution API versions:

- MUST be in form `<major>.<minor>` or `<major>`, where `<major>` is equivalent to `<major>.0`
- When `<major>` is greater than `0` increments to `<minor>` SHALL exclusively indicate additive changes
- Each Distributable artifact MUST contain the label `io.buildpacks.distribution.api` denoting the distribution API

## Artifact Format

### Buildpackage

A buildpackage is a buildpack packaged for distribution.

A buildpackage MUST exist in one of the following formats:

* An OCI Image on an image registry.
* An OCI Image in a Docker daemon.
* An uncompressed tar archive in [oci-layout](https://github.com/opencontainers/image-spec/blob/main/image-layout.md) format.
  * The file SHOULD have the extension `.cnb`.

[†](README.md#linux-only)For Linux buildpackages, all FS layers MUST be buildpack layers.

[‡](README.md#windows-only)For Windows buildpackages, all FS layers MUST be either buildpack or OS layers.

Each buildpack layer blob MUST contain a single [buildpack](./buildpack.md) at the following file path:

```
/cnb/buildpacks/<buildpack ID>/<buildpack version>/
```

If the buildpack ID contains a `/`, it MUST be replaced with `_` in the directory name.

A buildpack MUST contain a `buildpack.toml` file at its root directory.

#### Labels

The following labels MUST be set in the buildpack image(through the image config's `Labels` field):

| Label                                | Description                                                                                       |
| ------------------------------------ | ------------------------------------------------------------------------------------------------- |
| `io.buildpacks.buildpack.metadata` | A JSON object representing information about the packaged entrypoint buildpack                    |
| `io.buildpacks.buildpack.layers`   | A JSON object representing information about all packaged buildpacks and their associated layers. |

##### `io.buildpacks.buildpack.metadata` (JSON)

```json
{
  "id": "<buildpack ID (required)>",
  "name": "<buildpack name (optional)>",
  "version": "<buildpack version (required)>",
  "homepage": "<buildpack home page (optional)>",
}
```

##### `io.buildpacks.buildpack.layers` (JSON)

```json
{
  # schema of a meta-buildpack
  "<buildpack ID (required)>": {
    "<buildpack version (required)>": {
      "api": "<buildpack API (required)>",
      "order": [
        {
          "group": [
            {
              "id": "<inner buildpack ID (required)>",
              "version": "<inner buildpack version (required)>"
            }
          ]
        }
      ],
      "layerDiffID": "<diff ID of buildpack layer (required)>",
      "homepage": "<buildpack homepage (optional)>",
      "name": "<buildpack name (optional)>"
    }
  },
  # schema of a regular buildpack
  "<buildpack ID (required)>": {
    "<buildpack version (required)>": {
      "api": "<buildpack API (required)>",
      "layerDiffID": "<diff ID of buildpack layer (required)>",
      "homepage": "<buildpack homepage (optional)>",
      "name": "<buildpack name (optional)>"
    }
  }
}
```

For each buildpack layer, the buildpack ID and the buildpack version MUST be provided in `io.buildpacks.buildpack.layers`.

The buildpack ID and version MUST match a buildpack provided by a layer blob.

For a buildpackage to be valid, each `buildpack.toml` describing a buildpack implementation MUST have all listed targets.

### Lifecycle

The following defines how a `lifecycle` SHOULD be packaged for distribution. The `lifecycle` is the component that orchestrates buildpack execution, then assembles the resulting artifacts into a final app image.

The Lifecycle MUST exist in one of the following formats:

* An OCI Image on an image registry.
* An OCI Image in a Docker daemon.
* An uncompressed tar archive in [oci-layout](https://github.com/opencontainers/image-spec/blob/main/image-layout.md) format.

#### Filesystem

A lifecycle image MUST have the following directories/files

- `/cnb/lifecycle/<lifecycle binaries>` &rarr; An implementation of the lifecycle, which contains the required lifecycle binaries for [building images](https://github.com/buildpacks/spec/blob/main/platform.md#build).

#### Labels

| Label                             | Description                                                 |
| --------------------------------- | ----------------------------------------------------------- |
| `io.buildpacks.lifecycle.version` | A string, representing the semver version of the lifecycle. |
| `io.buildpacks.lifecycle.apis`    | A JSON object representing the APIs the lifecycle supports. |

##### `io.buildpacks.lifecycle.apis` (JSON)

```json
{
  "buildpack": {
    "deprecated": ["<list of versions>"],
    "supported": ["<list of versions>"]
  },
  "platform": {
    "deprecated": ["<list of versions>"],
    "supported": ["<list of versions>"]
  }
}
```

Where:

* `supported`:

  * contains an array of support API versions:
    * for versions `1.0+`, version `x.n` implies support for [`x.0`, `x.n`]
    * should be a superset of `deprecated`
    * should only contain APIs that correspond to a spec release
* `deprecated`:

  * contain an array of deprecated APIs:
    * should only contain `0.x` or major versions
    * should only contain APIs that correspond to a spec release

### Build Image

The following defines how a build image SHOULD be packaged for distribution as an OCI Image. The build image is the component that provides the base image from which the build environment is constructed.

The image configuration refers to the OCI Image configuration as mentioned [here](https://github.com/opencontainers/image-spec/blob/main/config.md#properties).

#### Image Configuration

The Build Image MUST contain the following configurations:

* Image Config's `config.User` field MUST be set to the user [†](README.md#operating-system-conventions)UID/[‡](README.md#operating-system-conventions)SID with a writable home directory.
* Image Config's `os` field MUST be set to the underlying operating system used by the build image.
* Image Config's `architecture` field MUST be set to the underlying operating system architecture used by the build image.
* Image Config's `variant` field MUST be set to the underlying architecture variant.

#### Environment Variables

The Build Image MUST contain the following Environment Variables:

* Image Config's `config.Env` field MUST have the environment variable `CNB_USER_ID` set to the user [†](README.md#operating-system-conventions)UID/[‡](README.md#operating-system-conventions)SID of the user specified in the `User` field.
* Image Config's `config.Env` field MUST have the environment variable `CNB_GROUP_ID` set to the primary group [†](README.md#operating-system-conventions)GID/[‡](README.md#operating-system-conventions)SID of the user specified in the `User` field.

#### Labels

The Build Image SHOULD contain the following Labels on the image configuration:

| Label                                | Description                                         |
| ------------------------------------ | --------------------------------------------------- |
| `io.buildpacks.distribution.name`    | A string denoting the operating system distribution |
| `io.buildpacks.distribution.version` | A string denoting the operating system version      |

### Run Image

The following defines how a run image SHOULD be packaged for distribution as an OCI Image. The run image is the component that provides the base from which app images are built.

The image configuration refers to the OCI Image configuration as mentioned [here](https://github.com/opencontainers/image-spec/blob/main/config.md#properties).

#### Image Configuration

The Run Image MUST contain the following configurations:

* Image Config's `config.User` field MUST be set to a non-root user with a writable home directory. The user SHOULD be a **DIFFERENT** user [†](README.md#operating-system-conventions)UID/[‡](README.md#operating-system-conventions)SID from that specified in the build image.
* Image Config's `os` field MUST be set to the underlying operating system used by the run image.
* Image Config's `architecture` field MUST be set to the underlying operating system used by the run image.

The Run Image SHOULD contain the following configurations:

* Image Config's `variant` field SHOULD be set to the underlying architecture variant.

#### Environment Variables

The Run Image MUST contain the following Environment Variables:

* Image Config's `config.Env` field MUST have the environment variable `CNB_USER_ID` set to the user [†](README.md#operating-system-conventions)UID/[‡](README.md#operating-system-conventions)SID of the user specified in the `User` field.
* Image Config's `config.Env` field MUST have the environment variable `CNB_GROUP_ID` set to the primary group GID/SID of the user specified in the `User` field.
* Image Config's `config.Env` field MUST have the environment variable `PATH` set to a valid set of paths or explicitly set to empty (`PATH=`).

#### Labels

The Run Image SHOULD contain the following Labels on the image configuration:

| Label                                | Description                                                 |
| ------------------------------------ | ----------------------------------------------------------- |
| `io.buildpacks.distribution.name`    | A string denoting the Operating System distribution         |
| `io.buildpacks.distribution.version` | A string denoting the Operating System distribution version |
| `io.buildpacks.id`                   | `<Target ID>`                                               |

Where,

`<Target ID>` is an identifier specified on the runtime image that MAY be used to apply target-specific logic.

### Builder

The following specifies the artifact format for a buildpacks builder.

A builder is an OCI Image that provides a distributable build environment.

A platform supporting builders SHOULD allow users to configure the build environment for a provided builder.

#### General Requirements

The builder image MUST contain an implementation of the [lifecycle](#lifecycle), and a [build-time](#build-image) environment, and MAY contain [buildpacks](#buildpackage). Platforms SHOULD use builders to ease the build process.

#### Filesystem

A builder MUST have the following directories/files:

* `/cnb/order.toml` &rarr; As defined in the [platform specification](https://github.com/buildpacks/spec/blob/main/platform.md#ordertoml-toml).

#### Environment Variables

A builder MUST be an extension of a Build Image, and MUST retain all the specified environment variables set on the original build image, as specified in the Build Image specifications.

The following environment variables MAY be set on the builder (through the image config's `Env` field):

| Env Variable           | Description                                                                        |
| ---------------------- | ---------------------------------------------------------------------------------- |
| `CNB_APP_DIR`          | Application directory of the build environment.                                    |
| `CNB_LAYERS_DIR`       | The directory to create and store `layers` in the build environment.               |
| `CNB_PLATFORM_DIR`     | The directory to create and store platform focused files in the build environment. |
| `SERVICE_BINDING_ROOT` | The directory where services are bound.                                            |

If any environment variable defined above is set on the builder, the specified path MUST be present and writable by the Build Image user.

#### Labels

A builder MUST be an extension of a Build Image, and MUST retain all the specified Labels set on the original build image, as specified in the Build Image specifications.

A builder image MUST contain an implementation of the [lifecycle](#lifecycle), and MUST retain all the specified Labels set on the original Lifecycle image, as specified in the lifecycle distribution specifications.

A builder image MAY contain buildpacks, and MAY retain all the specified Labels set on the original buildpackage, as specified in the [buildpackage](#buildpackage) specification with the following exceptions:

- `io.buildpacks.buildpack.metadata` MUST not be set.
- `io.buildpacks.buildpack.layers`  on the builder SHOULD be a merged version based on all buildpackages combined and thereby have of all packaged buildpacks represented.

In addition to all inherited labels above, the following labels MUST be set in the builder environment (through the image config's `Labels` field):

| Label                            | Description                                                   |
| -------------------------------- | ------------------------------------------------------------- |
| `io.buildpacks.builder.metadata` | A JSON object representing builder metadata.                  |
| `io.buildpacks.buildpack.order`  | A JSON object representation of the `/cnb/order.toml` file.   |

##### `io.buildpacks.builder.metadata` (JSON)

```json
{
  "description": "<string>",
  "createdBy": {
    "name": "<string>",
    "version": "<string>",
  }
}
```

Where:

- `description` (optional) is a human readable description of the builder.
- `createdBy` (optional) is information about the tool that created the builder.
  - `name` (optional) is the name of the tool that created the builder.
  - `version` (optional) is the version of the tool that created the builder.
