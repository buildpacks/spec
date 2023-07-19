# Distribution Specification

This document specifies the artifact format, delivery mechanism, and order resolution process for buildpacks.


## Table of Contents

<!-- Using https://github.com/yzhang-gh/vscode-markdown to manage toc -->
- [Distribution Specification](#distribution-specification)
  - [Table of Contents](#table-of-contents)
  - [Distribution API Version](#distribution-api-version)
  - [Artifact Format](#artifact-format)
    - [Buildpack](#buildpack)
    - [Buildpackage](#buildpackage)

## Distribution API Version

This document specifies Distribution API version `0.4`.

Distribution API versions:
 - MUST be in form `<major>.<minor>` or `<major>`, where `<major>` is equivalent to `<major>.0`
 - When `<major>` is greater than `0` increments to `<minor>` SHALL exclusively indicate additive changes
 - Each Distributable artifact MUST contain the label `io.buildpacks.distribution.api` denoting the distribution API

## Artifact Format

### Buildpack

A buildpack MUST contain a [`buildpack.toml`](buildpack.md#buildpacktoml-toml) file at its root directory.

### Buildpackage

A buildpackage MUST exist as either an OCI image on an image registry, an OCI image in a Docker daemon, or a `.cnb` file.

A `.cnb` file MUST be an uncompressed tar archive containing an OCI image. Its file name SHOULD end in `.cnb`.

[†](README.md#linux-only)For Linux buildpackages, all FS layers MUST be buildpack layers.

[‡](README.md#windows-only)For Windows buildpackages, all FS layers MUST be either buildpack or OS layers.

Each buildpack layer blob MUST contain a single buildpack at the following file path:

```
/cnb/buildpacks/<buildpack ID>/<buildpack version>/
```

If the buildpack ID contains a `/`, it MUST be replaced with `_` in the directory name.

A buildpack MUST contain a `buildpack.toml` file at its root directory.


A buildpack ID, buildpack version, and at least one stack MUST be provided in the OCI image config as a Label.

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

### Builder

The following specifies the artifact format for a buildpacks builder.

A builder is an OCI Image that provides a distributable build environment.

A platform supporting builders SHOULD allow users to configure the build environment for a provided builder.

A platform supporting builder creation MUST be able to create a valid builder from a valid [builder.toml](#builder.toml) file. (NOTE: much like (roses by any other name)[https://www.ncbi.nlm.nih.gov/pmc/articles/PMC543212], the platform MUST parse the contents of a builder-specifying toml file regardless of how it's called.)

#### General Requirements

The builder MUST contain an implementation of the [lifecycle](#lifecycle), and a [build-time](#build-image) environment, and MAY contain [buildpacks](#buildpackage). Platforms SHOULD use builders to encapsulate the build process.

#### Filesystem

A builder MUST have the following directories/files:

* `/cnb/order.toml` &rarr; As defined in the [platform specification](https://github.com/buildpacks/spec/blob/main/platform.md#ordertoml-toml).

#### Environment Variables

A builder MUST be an extension of a Build Image, and MUST retain all the specified environment variables set on the original build image, as specified in the Build Image specifications.

#### Labels

A builder MUST be an extension of a Build Image, and MUST retain all the specified Labels set on the original build image, as specified in the Build Image specifications.

A builder MUST contain an implementation of the [lifecycle](#lifecycle), and MUST retain all the specified Labels set on the original Lifecycle image, as specified in the lifecycle distribution specifications.

A builder MAY contain buildpacks, and MAY retain all the specified Labels set on the original buildpackage, as specified in the [buildpackage](#buildpackage) specification with the following exceptions:

- `io.buildpacks.buildpack.metadata` MUST not be set.
- `io.buildpacks.buildpack.layers`  on the builder MUST be a merged version based on all buildpackages combined and thereby have of all packaged buildpacks represented.

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

## Data Format

### Files

#### `builder.toml` (TOML)
```toml
description = "a human readable description output when you run `pack inspect-builder`"

[[buildpacks]]
id = "<buildpack ID>"
version = "<buildpack version>"
uri = "<uri>"

[[order]]
[[order.group]]
id = "<buildpack ID>"
version = "<buildpack version>"
optional = false

[build]
image = "build image"

[run]
[[run.images]]
image = "run image"
mirrors = "mirrors"

[lifecycle]
uri = "uri"
version = "ve.rs.ion"

```

Where:
- buildpacks list MAY be provided.
  - buildpacks.id and .version are optional but if provided MUST match an available buildpack in its buildpack.toml file.
  - buildpacks.uri MUST be provided.
- order list MUST be provided and MUST have at least one group list
  - order.group.id 
    - MUST be provided
    - SHOULD match an available buildpack in the buildpacks list
    - MUST be unique within a group
    - MAY be repeated within the order
  - order.group.version (optional) MAY be omitted if the ID alone is sufficient to identify a unique buildpack.
  - order.group.optional (optional) MUST be a boolean value specifying whether or not this buildpack is optional during detection.
- stack MAY be provided to platforms >= 0.12, but is deprecated in favor of `build` and `run` (see below). MUST be provided to platforms <= 0.11.
  - build-image MUST be included in a stack
  - run-image MUST be included
  - run-image-mirrors MAY be included
  - Deprecated stack example:
```
[stack]
id = "<stack id>"
build-image = "build image"
run-image = "run image"
run-image-mirrors = "mirrors"
```
- build SHOULD be provided to platforms >= 0.12, and MUST include an image
- run SHOULD be provided to platforms >= 0.12
  - run MUST have at least one `images` entry
  - each `run.images` MUST include an image
  - each `run.images` MAY include a list of mirrors (in the format `["mirror url", "mirror, url", ...]`)
- either stack or build and run images MUST be provided.
- lifecyle MAY be provided to specify a custom lifecycle - either uri or version (but not both) MUST be provided
  - uri MUST be a locator for a valid lifecycle image (mutually exclusive with version)
  - version MUST be a valid semver string (mutually exclusive with uri)


#### `package.toml` (TOML)

```(toml)
[buildpack]
uri = "uri"

[[buildpacks]]
uri = "uri"

[platform]
os = "operating system"

[[dependencies]]
uri = "uri"
image = "image uri"

```

Where:
- buildpack or buildpacks MUST be present, and MUST include a URI, which MAY be a relative or absolute path, or URL.
- platform MAY be provided to specify the operating system compatibility. If provided, `os` SHOULD be either `linux` (default) or `windows`
- dependencies list MAY be provided to specify URIs of order groups within a meta-buildpack.

