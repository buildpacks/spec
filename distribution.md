# Distribution Specification

This document specifies the artifact format, delivery mechanism, and order resolution process for buildpacks.


## Table of Contents

1. [Order Resolution](#order-resolution)
2. [Artifact Format](#artifact-format)
   1. [Buildpack Blob](#buildpack-blob)
   2. [Buildpackage](#buildpackage)
3. [Data Format](#data-format)
   1. [buildpack.toml (TOML)](#buildpack.toml-toml)

## Order Resolution

During detection, a buildpack ID or buildpack order definition MUST be resolved into individual buildpack implementations that include a `path` field.

The resolution process MUST follow this pattern:

Where:
- O and P are buildpack orders.
- A through H are buildpack implementations. 

Given:

<img src="http://tex.s2cms.ru/svg/%0AO%20%3D%0A%5Cbegin%7Bbmatrix%7D%0AA%2C%20%26%20B%20%5C%5C%0AC%2C%20%26%20D%0A%5Cend%7Bbmatrix%7D%0A" alt="
O =
\begin{bmatrix}
A, &amp; B \\
C, &amp; D
\end{bmatrix}
" />

<img src="http://tex.s2cms.ru/svg/%0AP%20%3D%0A%5Cbegin%7Bbmatrix%7D%0AE%2C%20%26%20F%20%5C%5C%0AG%2C%20%26%20H%0A%5Cend%7Bbmatrix%7D%0A" alt="
P =
\begin{bmatrix}
E, &amp; F \\
G, &amp; H
\end{bmatrix}
" />

We propose:

<img src="http://tex.s2cms.ru/svg/%0A%5Cbegin%7Bbmatrix%7D%0AE%2C%20%26%20O%2C%20%26%20F%0A%5Cend%7Bbmatrix%7D%20%3D%20%0A%5Cbegin%7Bbmatrix%7D%0AE%2C%20%26%20A%2C%20%26%20B%2C%20%26%20F%20%5C%5C%0AE%2C%20%26%20C%2C%20%26%20D%2C%20%26%20F%20%5C%5C%0A%5Cend%7Bbmatrix%7D%0A" alt="
\begin{bmatrix}
E, &amp; O, &amp; F
\end{bmatrix} = 
\begin{bmatrix}
E, &amp; A, &amp; B, &amp; F \\
E, &amp; C, &amp; D, &amp; F \\
\end{bmatrix}
" />

<img src="http://tex.s2cms.ru/svg/%0A%5Cbegin%7Bbmatrix%7D%0AO%2C%20%26%20P%0A%5Cend%7Bbmatrix%7D%20%3D%20%0A%5Cbegin%7Bbmatrix%7D%0AA%2C%20%26%20B%2C%20%26%20E%2C%20%26%20F%20%5C%5C%0AA%2C%20%26%20B%2C%20%26%20G%2C%20%26%20H%20%5C%5C%0AC%2C%20%26%20D%2C%20%26%20E%2C%20%26%20F%20%5C%5C%0AC%2C%20%26%20D%2C%20%26%20G%2C%20%26%20H%20%5C%5C%0A%5Cend%7Bbmatrix%7D%0A" alt="
\begin{bmatrix}
O, &amp; P
\end{bmatrix} = 
\begin{bmatrix}
A, &amp; B, &amp; E, &amp; F \\
A, &amp; B, &amp; G, &amp; H \\
C, &amp; D, &amp; E, &amp; F \\
C, &amp; D, &amp; G, &amp; H \\
\end{bmatrix}
" />

Note that buildpack IDs are expanded depth-first in left-to-right order.

If a buildpack order entry within a group has the parameter `optional = true`, then a copy of the group without the entry MUST be repeated after the original group.

## Artifact Format

### Buildpack Blob

A buildpack blob MUST contain one or more buildpacks.

A buildpack blob MUST be packaged as gzip-compressed tarball.
Its filename should end in `.tgz`.

A buildpack blob MUST contain a `buildpack.toml` file at its root directory.

A buildpack defined within `buildpack.toml` MUST either be:

1. A buildpack implementation specified by a `stacks` field and optionally a `path` field or
2. A buildpack order specyfied by an `order` field referencing other buildpack IDs/versions.

For each buildpack definition, it is OPTIONAL for buildpacks defined in `[[buildpacks.order]]` to be included in the buildpack blob.

### Buildpackage

A buildpackage MUST exist as either an OCI image on an image registry, an OCI image in a Docker daemon, or a `.cnb` file.

A `.cnb` file MUST be an uncompressed tar archive containing an OCI image. Its file name SHOULD end in `.cnb`.

Each FS layer blob in the buildpackage MUST contain a single buildpack blob and at least one symlink.

A symlink MUST be created for each buildpack version that the blob is assembled to support.

```
/cnb/blobs/<sha256 checksum of buildpack blob tgz>/
/cnb/by-id/<buildpack ID1>/<buildpack version> -> /cnb/blobs/<sha256 checksum of blob tgz>/
/cnb/by-id/<buildpack ID2>/<buildpack version> -> /cnb/blobs/<sha256 checksum of blob tgz>/
...
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

For a buildpackage to be valid, each buildpack implementation entry in each `buildpack.toml` MUST have all listed stacks.

For each listed stack, all associated buildpacks MUST be a candidate for detection when the entrypoint buildpack ID and version are selected.

Each stack ID MUST only be present once.
For a given stack, the `mixins` list MUST enumerate mixins such that no included buildpacks are missing a mixin for the stack.

Fewer stack entries as well as additional mixins for a stack entry MAY be specified.

## Data Format

### buildpack.toml (TOML)

```toml
[[buildpacks]]
id = "<buildpack ID>"
name = "<buildpack name>"
version = "<buildpack version>"
path = "<path to buildpack>"

[[buildpacks.order]]
[[buildpacks.order.group]]
id = "<buildpack ID>"
version = "<buildpack version>"
optional = false

[[buildpacks.stacks]]
id = "<stack ID>"
mixins = ["<mixin name>"]
build-images = ["<build image tag>"]
run-images = ["<run image tag>"]

[buildpacks.metadata]
# buildpack-specific data
```

If an order is specified, then `path` and `stacks` MUST not be specified.
A buildpack path MUST default to `.` when not specified and when `order` is not specified.

Buildpack authors MUST choose a globally unique ID, for example: "io.buildpacks.ruby".

The buildpack ID:
- MUST only contain numbers, letters, and the characters `.`, `/`, and `-`.
- MUST NOT be `config` or `app`.
- MUST NOT be identical to any other buildpack ID when using a case-insensitive comparison.

Stack authors MUST choose a globally unique ID, for example: "io.buildpacks.mystack".

The stack ID:
- MUST only contain numbers, letters, and the characters `.`, `/`, and `-`.
- MUST NOT be identical to any other stack ID when using a case-insensitive comparison.

The stack `build-images` and `run-images` are suggested sources of images for platforms that are unaware of the stack ID. Buildpack authors MUST ensure that these images include all mixins specified in `mixins`.
