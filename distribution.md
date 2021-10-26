# Distribution Specification

This document specifies the artifact format and the delivery mechanism for the buildpacks core components.


## Table of Contents

<!-- Using https://github.com/yzhang-gh/vscode-markdown to manage toc -->
- [Distribution Specification](#distribution-specification)
  - [Table of Contents](#table-of-contents)
  - [Distribution API Version](#distribution-api-version)
  - [Artifact Format](#artifact-format)
    - [Buildpackage](#buildpackage)

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

For each buildpack layer, the buildpack ID and the buildpack version MUST be provided in `io.buildpacks.buildpack.layers`

The following labels MUST be set in the buildpack image(through the image config's `Labels` field):

| Label             | Description | 
| --------          | -------- 
| `io.buildpacks.buildpack.metadata`     | A JSON object representing Buildpack Metadata   |
| `io.builpacks.buildpack.layers`| A JSON object representing the buildpack layers |


`io.buildpacks.buildpackage.metadata` (JSON)
```json
{
  "id": "<entrypoint buildpack ID>",
  "name": "<buildpack name>",
  "version": "<entrypoint buildpack version>",
  "homepage": "<buildpack home page",
}
```

`io.buildpacks.buildpack.layers` (JSON)
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
