# Platform Interface Specification

This document specifies the interface between a lifecycle and a platform.

A platform orchestrates a lifecycle to make buildpack functionality available to end-users such as application developers.

Examples of a platform might include:

1. A local CLI tool that uses buildpacks to create OCI images
2. A plugin for a continuous integration service that uses buildpacks to create OCI images
3. A cloud application platform that uses buildpacks to build source code before deployment

## Table of Contents

1. [Stacks](#stacks)
   1. [Build Image](#build-image)
   2. [Run Image](#run-image)
2. [Detection Order](#detection-order)
    
## Stacks

A stack consists of:
- A globally unique ID
- A tag reference to a build OCI image
- A tag reference to a run OCI image

A particular version of a stack consists of:
- A globally unique ID
- A version identifier
- A SHA reference to a build OCI image
- A SHA reference to a run OCI image

Both images MUST have a user,
- Named "pack",
- With UID 1000 ,
- With a home directory created at `/home/pack`,
- With a primary group named "pack" with GID 1000, and
- With no password set on the user or primary group.



### Build Image

The platform MUST execute the detection and build phases of the lifecycle on the build image.

The build image MUST specify a non-root user with an empty home directory at `$HOME` during the build phase.

The build image MUST have `ENTRYPOINT` set to a lifecycle component such that arguments may be provided via `CMD`.
This lifecycle component should accept the following arguments:
- TDB

A buildpack should support one or more stacks explicitly by stack ID.
A buildpack should use `$PACK_STACK_ID` during the detection and build phases to determine what dependencies it provides.

Stack authors SHOULD ensure that build image versions maintain ABI-compatibility with previous versions, although violating this requirement will not change the behavior of previously built images containing app and launch layers.

### Run Image

The platform MUST provide the lifecycle with a reference to the run image during the export phase.

The run image MUST specify a non-root user with an empty home directory at `$HOME` during launch.

The run image MUST have `ENTRYPOINT` set to a lifecycle component such that arguments may be provided via `CMD`.
This lifecycle component should accept the following arguments:
- TDB

Stack authors MUST ensure that new run image versions maintain [ABI-compatibility](https://en.wikipedia.org/wiki/Application_binary_interface) with previous versions.

App and launch layers MUST NOT change behavior when the run image layers are upgraded to newer versions, unless those behavior changes are intended to fix security vulnerabilities.



## Detection Order

The platform MUST provide the lifecycle with a list of buildpack groups for the detection phase.

This list of buildpack groups MUST be provided in Order Definition format. Example:

```toml
[[groups]]
buildpacks = [
  { id = "sh.packs.buildpacks.nodejs", version = “latest”, optional = true },
  { id = "sh.packs.buildpacks.dotnet-core", version = “latest” }
]

[[groups]]
buildpacks = [
  { id = "sh.packs.buildpacks.nodejs", version = “latest”, optional = true },
  { id = "sh.packs.buildpacks.ruby", version = “latest” }
]

[[groups]]
buildpacks = [
  { id = "sh.packs.buildpacks.python", version = “latest” },
  { id = "sh.packs.buildpacks.ruby", version = “latest” }
]
```

## Build

User-provided environment variables intended for build and launch SHOULD NOT come from the same list.
The user SHOULD be encouraged to define them separately.
The platform MAY determine the initial build-time and runtime environment.
The lifecycle MUST NOT assume that all platforms provide an identical environment.

## Data Format

### Order Definition (TOML)

```toml
[[groups]]

buildpacks = [
  {id = "<buildpack ID>", version = "<buildpack version>", optional = <:bool>}
 ]
```

### Image Layers: Launch

Where <id> is a globally-unique buildpack identifier.

Where <layer> is an identifier chosen by the buildpack representing the contents of the layer

**Image FS Layers (for each layer for each buildpack)**

/launch/<id>/<layer>/		contains: <launch>/<layer>/

**Image FS Layer**

/launch/app/			contains: <app>/

**Image FS Layer**

/launch/config/			contains: metadata.toml (for launch)

**Image Config Layer**

LABEL				buildpack names and versions

LABEL				layer metadata for each layer

### Image Layers: Cache

Where <id> is a globally-unique buildpack identifier.

Where <layer> is an identifier chosen by the buildpack representing the contents of the layer

**Image FS Layers (for each layer for each buildpack)**

/cache/<id>/<layer>/		contains: <cache>/<layer>/

**Mount or Ephemeral Copy**

/launch/app/			contains: <app>/

Note 1: test/development cache is stored separately from the sluglet cache.

Note 2: cache may or may not be stored in a registry

## Runtime Layer Rebasing

Runtime layer rebasing allows for O(CF) or O(Heroku) performance when updating OS-level dependencies, with regard to data transfer.
When a new run base image is available, the first sluglet layer is rebased on the new run base image with remote image metadata manipulation.
Similarly, individual sluglet layers MAY be replaced by successive remote rebase and/or append operations.

## Security Model

Implementations of this specification SHOULD run detection steps as well as the inner execution steps (2-5, which do not require reading or writing to an image repository) in a separate container (or at least a separate process space) that does not include credentials for access to any of the resources defined in this document.
Neither buildpack code nor app code SHOULD be trusted with these types of credentials.

## Layer Caching

The purpose of layer caching is to
1. Minimize the execution time of the build phase.
2. Minimize persistent disk usage. 

This is achieved by
1. Reducing the number of build operations.
2. Reducing data transfer. 
3. Enabling de-duplication of stored image layers.
