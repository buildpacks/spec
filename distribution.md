# Distribution Specification

This document specifies the artifact format and the delivery mechanism for the buildpacks core components.


## Table of Contents

<!-- Using https://github.com/yzhang-gh/vscode-markdown to manage toc -->
- [Distribution Specification](#distribution-specification)
  - [Table of Contents](#table-of-contents)
  - [Distribution API Version](#distribution-api-version)
  - [Artifact Format](#artifact-format)
    - [Buildpackage](#buildpackage)
    - [Lifecycle](#lifecycle)

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

A buildpack MUST contain a `buildpack.toml` file at its root directory.

#### Labels

For each buildpack layer, the buildpack ID and the buildpack version MUST be provided in `io.buildpacks.buildpackage.layers`

The following labels MUST be set in the buildpack image(through the image config's `Labels` field):

| Label             | Description | 
| --------          | -------- 
| `io.buildpacks.buildpackage.metadata`     | A JSON object representing Buildpack Metadata   |
| `io.builpacks.buildpackage.layers`| A JSON object representing the buildpack layers |


`io.buildpacks.buildpackage.metadata` (JSON)
```json
{
  "id": "<entrypoint buildpack ID>",
  "name": "<buildpack name>",
  "version": "<entrypoint buildpack version>",
  "homepage": "<buildpack home page",
}
```

`io.buildpacks.buildpackage.layers` (JSON)
```json
{
  "<buildpack ID>": {
    "<buildpack version>": {
      "api": "<buildpack API>",
      "order": [
        {
          "group": [
            {
              "id": "<inner buildpack ID>",
              "version": "<inner buildpack version>"
            }
          ]
        }
      ],
      "layerDiffID": "<diff-ID>",
      "homepage": "<buildpack homepage>",
      "name": "<buildpack name>",
    }
  },
  "<inner buildpack>": {
    "<inner buildpack version>": {
      "api": "<buildpack API>",
      "layerDiffID": "<diff-ID>",
      "homepage": "<buildpack homepage>",
      "name": "<buildpack name>",      
    }
  }
}
```

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

| Label             | Description
| --------          | --------
| `io.buildpacks.lifecycle.version`  | A string, representing the semver version of the lifecycle.
| `io.buildpacks.lifecycle.apis`     | A JSON object representing the APIs the lifecycle supports.

`io.buildpacks.lifecycle.apis` (JSON)

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
