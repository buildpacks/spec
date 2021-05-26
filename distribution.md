# Distribution Specification

This document specifies the artifact format, delivery mechanism, and order resolution process for buildpacks.


## Table of Contents

<!-- Using https://github.com/yzhang-gh/vscode-markdown to manage toc -->
- [Distribution Specification](#distribution-specification)
  - [Table of Contents](#table-of-contents)
  - [Distribution API Version](#distribution-api-version)
  - [Artifact Format](#artifact-format)
    - [Asset Package](#asset-package)
    - [Buildpack](#buildpack)
    - [Buildpackage](#buildpackage)

## Distribution API Version

This document specifies Distribution API version `0.2`.

Distribution API versions:
 - MUST be in form `<major>.<minor>` or `<major>`, where `<major>` is equivalent to `<major>.0`
 - When `<major>` is greater than `0` increments to `<minor>` SHALL exclusively indicate additive changes

## Artifact Format

### Asset Package 

An Asset Package MUST exist as and OCI image on a registry, and OCI image in a Docker daemon or as an uncompressed tar archive containing an OCI image.

- For Linux asset packages, all FS layers MUST contain one or more assets.
    
- For Windows asset packages, all FS layers MUST be an OS layer or either contain one or more assets.

FS asset layers MUST contain only asset files at the following file path:

```
/cnb/assets/<asset-sha256>
```

where `asset-sha256` is the `sha256` digest of the asset file.

In the resulting asset package image all layers MUST be alphabetically sorted by diffID to allow for reproducability.

#### Label

Asset Package Image must contain the following two labels with the following contents:

`io.buildpacks.asset.layers`
{
  "name": "asset-package-org/asset-package-name"
}

Each asset in an asset package image must have an entry in the below label.
`io.buildpacks.asset.metadata`:
```
{
  "<asset-sha256>": {
    "name": "(optional)",
    "id": "(required)",
    "version": "(required)",
    "layerDiffID": "(required)",
    "uri": "(optional)",
    "licenses" : ["(optional)"],
    "description" : "(optional)",
    "homepage" : "(optional)",
    "stacks": [
      "(optional)",
      "..."
    ],
    "metadata": {
      "(optional)": "(optional)"
    }
  }
}
```
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

A buildpack ID, buildpack version, and at least one stack MUST be provided in the OCI image config as a Label.

Label: `io.buildpacks.buildpackage.metadata`
```json
{
  "id": "<entrypoint buildpack ID>",
  "version": "<entrypoint buildpack version>",
  "stacks": [
    {
      "id": "<stack ID>",
      "mixins": ["<mixin name>"]
    }
  ]
}
```

The buildpack ID and version MUST match a buildpack provided by a layer blob.

For a buildpackage to be valid, each `buildpack.toml` describing a buildpack implementation MUST have all listed stacks.

For each listed stack, all associated buildpacks MUST be a candidate for detection when the entrypoint buildpack ID and version are selected.

Each stack ID MUST only be present once.
For a given stack, the `mixins` list MUST enumerate mixins such that no included buildpacks are missing a mixin for the stack.

Fewer stack entries as well as additional mixins for a stack entry MAY be specified.
