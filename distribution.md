# Distribution Specification

This document specifies the artifact format, delivery mechanism, and order resolution process for buildpacks.


## Table of Contents

1. [Order Resolution](#order-resolution)
2. [Artifact Format](#artifact-format)
   1. [Buildpack](#buildpack)
   2. [Buildpackage](#buildpackage)
3. [Data Format](#data-format)
   1. [buildpack.toml (TOML)](#buildpack.toml-toml)

## Order Resolution

During detection, a buildpack ID or buildpack order definition MUST be resolved into individual buildpack implementations.

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

### Buildpack

A buildpack MUST contain a `buildpack.toml` file at its root directory.

`buildpack.toml` MUST either contain:

1. A buildpack implementation denoted by the presence of a `stacks` field or
2. A buildpack order specified by an `order` field referencing other buildpack IDs/versions.

### Buildpackage

A buildpackage MUST exist as either an OCI image on an image registry, an OCI image in a Docker daemon, or a `.cnb` file.

A `.cnb` file MUST be an uncompressed tar archive containing an OCI image. Its file name SHOULD end in `.cnb`.

Each FS layer blob in the buildpackage MUST contain a single buildpack at the following file path:

```
/cnb/by-id/<buildpack ID>/<buildpack version>/
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

## Data Format

### buildpack.toml (TOML)

```toml
[buildpack]
id = "<buildpack ID>"
name = "<buildpack name>"
version = "<buildpack version>"
clear-env = false

[[order]]
[[order.group]]
id = "<buildpack ID>"
version = "<buildpack version>"
optional = false

[[stacks]]
id = "<stack ID>"
mixins = ["<mixin name>"]
build-images = ["<build image tag>"]
run-images = ["<run image tag>"]

[metadata]
# buildpack-specific data
```

If an order is specified, then `stacks` MUST not be specified.

A buildpack reference inside of a `group` MUST contain an `id` and `version`.

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
