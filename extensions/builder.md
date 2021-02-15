# Builder

A **builder** is an image that MUST contain an implementation of the lifecycle, and build-time environment, and MAY contain buildpacks. Platforms SHOULD use builders to ease the build process.

## Table of Contents

## Required File/Directories
A builder MUST have the following directories/files:
- `/cnb/order.toml` &rarr; As defined in the [platform specification][order-toml-spec]
- `/cnb/stack.toml` &rarr; As defined in the [platform specification][stack-toml-spec]
- `/cnb/lifecycle/<lifecycle binaries>` &rarr; An implementation of the lifecycle, which contains the required lifecycle binaries for [building images][lifecycle-for-build].

In addition, every buildpack blob contained on a builder MUST be stored at the following file path:
- `/cnb/buildpacks/...<buildpack ID>/<buildpack version>/`

## Environment Variables/Labels
A builder's environment is the build-time environment of the stack, and as such, MUST fulfill all of the [build image specifications][build-image-specs].

The following variables MUST be set in the builder environment (through the image config's `Env` field):

| Env Variable    | Description
|-----------------|--------------------------------------
| `WorkingDir`  | The working directory where buildpacks should default (eg: `/workspace`)
| `CNB_APP_DIR`  | Application directory of the build environment (eg: `/app`)
| `CNB_LAYERS_DIR`  | The directory to create and store `layers` in the build environment (eg: `/layers`)
| `CNB_PLATFORM_DIR`  | The directory to create and store platform focused files in the build environment (eg: `/platform`)

The following variables MAY be set in the builder environment (through the image config's `Env` field):

| Env Variable    | Description
|-----------------|--------------------------------------
| `SERVICE_BINDING_ROOT`  | The directory where services are bound

The following labels MUST be set in the builder environment (through the image config's `Labels` field):

| Env Variable    | Description
|-----------------|--------------------------------------
| `io.buildpacks.builder.api`  | The Builder API this builder implements (at this point, **0.1**)
| `io.buildpacks.builder.metadata`  | A JSON object representing [Builder Metadata](#iobuildpacksbuildermetadata)
| `io.buildpacks.buildpack.order`  | A JSON object representing the [order of the buildpacks stored on the builder](#iobuildpacksbuildpackorder)
| `io.buildpacks.buildpack.layers`  | A JSON object representing the [buildpack layers](#iobuildpacksbuildpacklayers)
| `io.buildpacks.lifecycle.version`  | A string, representing the version of the lifecycle stored in the builder (greater than `0.9.0`)
| `io.buildpacks.lifecycle.apis`  | A JSON object representing the [lifecycle APIs](#iobuildpackslifecycleapis) the lifecycle stored in the builder supports

### `io.buildpacks.builder.metadata`
The `io.buildpacks.builder.metadata` data should look like:
```json
{
  "description": "<Some description>",
  "stack": {
    "runImage": {
      "image": "<some/run-image>",
      "mirrors": [
        "<gcr.io/some/default>"
      ]
    }
  },
  "buildpacks": [
    {
      "id": "<buildpack id>",
	  "version": "<buildpack version>",
	  "homepage": "http://geocities.com/top-bp"
	}
  ],
  "createdBy": {
    "name": "<creator of builder>",
    "version": "<version of tool used to create builder>"
  }
}
```

### `io.buildpacks.buildpack.order`
The `io.buildpacks.buildpack.order` data MUST look like:
```json
[
	{
	  "group":
		[
		  {
			"id": "<buildpack id>",
			"version": "<buildpack version>",
			"optional": "<bool>"
		  }
		]
	}
]
```

### `io.buildpacks.buildpack.layers`
The `io.buildpacks.buildpack.layers` data should look like:
```json
{
  "<buildpack id>": {
    "<buildpack version>": {
      "api": "<buildpack API>",
      "order": [
        {
          "group": [
            {
              "id": "<inner buildpack>",
              "version": "<inner bulidpacks version>"
            }
          ]
        }
      ],
      "layerDiffID": "sha256:test.nested.sha256",
	  "homepage": "http://geocities.com/top-bp"
    }
  },
  "<inner buildpack>": {
    "<inner buildpacks version>": {
      "api": "<buildpack API>",
      "stacks": [
        {
          "id": "<stack ID buildpack supports>",
          "mixins": ["<list of mixins required>"]
        }
      ],
      "layerDiffID": "sha256:test.bp.one.sha256",
	  "homepage": "http://geocities.com/cool-bp"
    }
  }
}
```

### `io.buildpacks.lifecycle.apis`
The `io.buildpacks.lifecycle.apis` data should look like:
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
[order-toml-spec]: https://github.com/buildpacks/spec/blob/main/platform.md#ordertoml-toml
[stack-toml-spec]: https://github.com/buildpacks/spec/blob/main/platform.md#stacktoml-toml
[lifecycle-for-build]: https://github.com/buildpacks/spec/blob/main/platform.md#build
