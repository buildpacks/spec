# Builder Specification <!-- omit in toc -->

This document specified the artifact format for a builder.

A builder is an OCI image that provides a distributable build environment.

A [platform][platform-spec] supporting the builder extension specification SHOULD allow users to configure the build environment with a provided builder.

## Table of Contents <!-- omit in toc -->
- [General Requirements](#general-requirements)
  - [Builder API Version](#builder-api-version)
  - [File/Directories](#filedirectories)
  - [Environment Variables](#environment-variables)
  - [Labels](#labels)
- [Data Format](#data-format)
  - [Labels](#labels-1)
    - [`io.buildpacks.builder.metadata` (JSON)](#iobuildpacksbuildermetadata-json)
    - [`io.buildpacks.buildpack.order` (JSON)](#iobuildpacksbuildpackorder-json)
    - [`io.buildpacks.buildpack.layers` (JSON)](#iobuildpacksbuildpacklayers-json)
    - [`io.buildpacks.lifecycle.apis` (JSON)](#iobuildpackslifecycleapis-json)

## General Requirements
A **builder** is an image that MUST contain an implementation of the lifecycle, and build-time environment, and MAY contain buildpacks. Platforms SHOULD use builders to ease the build process.

### Builder API Version
This document specifies Builder API version `0.1`.

Builder API versions:
- MUST be in form `<major>.<minor>` or `<major>`, where `<major>` is equivalent to `<major>.0`
- When `<major>` is greater than `0` increments to `<minor>` SHALL exclusively indicate additive changes

### File/Directories
A builder MUST have the following directories/files:
- `<CNB_BUILDPACKS_DIR>/order.toml` &rarr; As defined in the [platform specification][order-toml-spec]
- `<CNB_BUILDPACKS_DIR>/stack.toml` &rarr; As defined in the [platform specification][stack-toml-spec]
- `<CNB_BUILDPACKS_DIR>/lifecycle/<lifecycle binaries>` &rarr; An implementation of the lifecycle, which contains the required lifecycle binaries for [building images][lifecycle-for-build].

In addition, every buildpack blob contained on a builder MUST be stored at the following file path:
- `<CNB_BUILDPACKS_DIR>/buildpacks/...<buildpack ID>/<buildpack version>/`

If the buildpack ID contains a `/`, it MUST be replaced with `_` in the directory name.

All specified files and directories are writeable by the build environment's User. 

### Environment Variables
A builder MUST be an extension of a build-image, and MUST retain all specified environment variables and labels set on the original build image, as specified in the [platform specifications][build-image-specs].

The following environment variables MUST be set on the builder:

| Env Variable       | Description                                                                                         |
| ------------------ | --------------------------------------------------------------------------------------------------- |
| `CNB_APP_DIR`      | Application directory of the build environment (eg: `/workspace`)                                   |
| `CNB_LAYERS_DIR`   | The directory to create and store `layers` in the build environment (eg: `/layers`)                 |
| `CNB_PLATFORM_DIR` | The directory to create and store platform focused files in the build environment (eg: `/platform`) |

The following variables MAY be set in the builder environment (through the image config's `Env` field):

| Env Variable           | Description                            | Default |
| ---------------------- | -------------------------------------- | ---- |
| `SERVICE_BINDING_ROOT` | The directory where services are bound | - |
| `CNB_BUILDPACKS_DIR` | The directory where CNB required files are | `/cnb` |

### Labels
The following labels MUST be set in the builder environment (through the image config's `Labels` field):

| Label                             | Description                                                                                                              |
| --------------------------------- | ------------------------------------------------------------------------------------------------------------------------ |
| `io.buildpacks.builder.api`       | The [Builder API Version](#builder-api-version) this builder implements                                                  |
| `io.buildpacks.builder.metadata`  | A JSON object representing [Builder Metadata](#iobuildpacksbuildermetadata)                                              |
| `io.buildpacks.buildpack.order`   | A JSON object representing the [order of the buildpacks stored on the builder](#iobuildpacksbuildpackorder)              |
| `io.buildpacks.buildpack.layers`  | A JSON object representing the [buildpack layers](#iobuildpacksbuildpacklayers)                                          |
| `io.buildpacks.lifecycle.version` | A string, representing the version of the lifecycle stored in the builder                                                |
| `io.buildpacks.lifecycle.apis`    | A JSON object representing the [lifecycle APIs](#iobuildpackslifecycleapis) the lifecycle stored in the builder supports |

## Data Format
### Labels
#### `io.buildpacks.builder.metadata` (JSON)

```json
{
  "description": "<description>",
  "stack": {
    "runImage": {
      "image": "<run image>",
      "mirrors": [
        "<run image mirror>"
      ]
    }
  },
  "buildpacks": [
    {
      "id": "<buildpack ID>",
	  "version": "<buildpack version>",
	  "homepage": "<buildpack homepage>"
	}
  ],
  "createdBy": {
    "name": "<tool name>",
    "version": "<tool version>"
  }
}
```

The `createdBy` metadata is meant to contain the name and version of the tool that created the builder. 

#### `io.buildpacks.buildpack.order` (JSON)

```json
[
	{
	  "group":
		[
		  {
			"id": "<buildpack ID>",
			"version": "<buildpack version>",
			"optional": false
		  }
		]
	}
]
```

#### `io.buildpacks.buildpack.layers` (JSON)

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
	  "homepage": "<buildpack homepage>"
    }
  },
  "<inner buildpack>": {
    "<inner buildpack version>": {
      "api": "<buildpack API>",
      "stacks": [
        {
          "id": "<stack ID buildpack supports>",
          "mixins": ["<list of mixins required>"]
        }
      ],
      "layerDiffID": "<diff-ID>",
	  "homepage": "<buildpack homepage>"
    }
  }
}
```

#### `io.buildpacks.lifecycle.apis` (JSON)

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

[//]: <> (Links)
[build-image-specs]: https://github.com/buildpacks/spec/blob/main/platform.md#build-image
[platform-spec]: https://github.com/buildpacks/spec/blob/main/platform.md
[order-toml-spec]: https://github.com/buildpacks/spec/blob/main/platform.md#ordertoml-toml
[stack-toml-spec]: https://github.com/buildpacks/spec/blob/main/platform.md#stacktoml-toml
[lifecycle-for-build]: https://github.com/buildpacks/spec/blob/main/platform.md#build
