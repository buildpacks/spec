# Platform Interface Specification

This document specifies the interface between a lifecycle and a platform.

A platform orchestrates a lifecycle to make buildpack functionality available to end-users such as application developers.

Examples of a platform might include:

1. A local CLI tool that uses buildpacks to create OCI images
2. A plugin for a continuous integration service that uses buildpacks to create OCI images
3. A cloud application platform that uses buildpacks to build source code before deployment

## Table of Contents

1. [Stacks](#stacks)
   1. [Compatibility Guarantees](#compatibility-guarantees)
   2. [Build Image](#build-image)
   3. [Run Image](#run-image)
2. [Buildpacks](#buildpacks)
   1. [Buildpacks Directory Layout](#buildpacks-directory-layout)
3. [Security Considerations](#security-considerations)
4. [Additional Guidance](#additional-guidance)
   1. [Run Layer Rebasing](#run-layer-rebasing)
   2. [Caching](#caching)
5. [Data Format](#data-format)
   1. [order.toml (TOML)](#order.toml-(toml))
   2. [group.toml (TOML)](#group.toml-(toml))

## Stacks

A "stack" refers to:

- A globally unique ID containing at least one period (`.`),
- A set of fully-qualified OCI image tags of build images, and
- A set of fully-qualified OCI image tags of run images.

A "stack version" refers to:

- A globally unique ID containing at least one period (`.`),
- A version identifier,
- The image SHA of a build OCI image, and
- The image SHA of a run OCI image.

### Compatibility Guarantees

Stack authors SHOULD ensure that build image versions maintain [ABI-compatibility](https://en.wikipedia.org/wiki/Application_binary_interface) with previous versions, although violating this requirement will not change the behavior of previously built images containing app and launch layers.

Stack authors MUST ensure that new run image versions maintain [ABI-compatibility](https://en.wikipedia.org/wiki/Application_binary_interface) with previous versions.
Stack authors MUST ensure that app and launch layers do not change behavior when the run image layers are upgraded to newer versions, unless those behavior changes are intended to fix security vulnerabilities.

### Build Image

The platform MUST execute the detection and build phases of the lifecycle on the build image.

The build image MUST specify a non-root user with an empty home directory at `$HOME` during the build phase.

The build image MUST have the stack ID set in the environment variable `PACK_STACK_ID` as well as using the label `io.buildpacks.stack.id`.

The build image MUST initiate the detection phase when the `/lifecycle/detector` executable is invoked.
Invoking this executable with no flags is equivalent to invoking it with all accepted flags and their default values, as such:

```bash
/lifecycle/detector -buildpacks /buildpacks -order /buildpacks/order.toml -group ./group.toml -plan ./plan.toml
```

Where:

- `-buildpacks` MUST specify input from a buildpacks directory as defined in the Buildpacks Directory Layout section.
- `-order` MUST specify input from an overriding `order.toml` file path as defined in the Data Format section.
- `-group` MUST specify output to a `group.toml` file path as defined in the Data Format section.
- `-plan` MUST specify output to a Build Plan as defined in the Buildpack Interface Specification.

The build image MUST initiate the build phase when the `/lifecycle/builder` executable is invoked.
Invoking this executable with no flags is equivalent to invoking it with all accepted flags and their default values, as such:

```bash
/lifecycle/builder -buildpacks /buildpacks -group ./group.toml -plan ./plan.toml
```

Where:

- `-buildpacks` MUST specify input from a buildpacks directory as defined in the Buildpacks Directory Layout section.
- `-group` MUST specify input from a `group.toml` file path as defined in the Detection Order section.
- `-plan` MUST specify input from a Build Plan as defined in the Buildpack Interface Specification.

### Run Image

The platform MUST provide the lifecycle with a reference to the run image during the export phase.

The run image MUST use a non-root user with an empty home directory at `$HOME` to execute the application process.

The build image MUST have the stack ID set using the label `io.buildpacks.stack.id`.

## Buildpacks

### Buildpacks Directory Layout

The buildpacks directory MUST contain unpackaged buildpacks such that:

- Each top-level directory is a buildpack ID.
- Each second-level directory is a buildpack version.
- Each top-level directory contains a `latest` symbolic link, which MUST point to the latest buildpack version.

Additionally, there MUST be an `order.toml` file at the root containing a list of buildpacks groups to use during the detection phase.

## Security Considerations

The platform SHOULD run each phase of the lifecycle in an isolated container to prevent untrusted app and buildpack code from accessing storage credentials needed during the export and analysis phases.
A more thorough explanation is provided in the Buildpack Interface Specification.

## Additional Guidance

User-provided environment variables intended for build and launch SHOULD NOT come from the same list.
The end-user SHOULD be encouraged to define them separately.
The platform MAY determine the initial build-time and runtime environment.
The lifecycle MUST NOT assume that all platforms provide an identical environment.

### Run Image Rebasing

Run image rebasing allows for fast stack updates for already-exported OCI images with minimal data transfer.
When a new stack version is available, the app and launch layers SHOULD be rebased on the new run image with remote OCI image metadata manipulation.

### Caching

Each platform SHOULD implement caching so as to appropriately optimize performance. Cache locality and availability MAY vary between platforms.

## Data Format

### order.toml (TOML)

```toml
[[groups]]

buildpacks = [
  {id = "<buildpack ID>", version = "<buildpack version>", optional = <bool>, source = "<URI>" }
 ]
```

Where:

- The buildpack ID MUST be present for each buildpack object in a group.
- The buildpack version MUST default to "latest" if not provided.
- Each buildpack MUST default to not optional if not specified in the object.
- Each buildpack object MAY specify a URI as optional metadata for the platform.

Example:

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
  { id = "sh.packs.buildpacks.ruby", version = “latest”, source = "https://example.com/ruby.tgz" }
]
```

### group.toml (TOML)

```toml
buildpacks = [
  { id = "<buildpack ID>", version = "<buildpack version>" }
 ]
```

Where:

- The buildpack ID MUST be present for each buildpack object in a group.
- The buildpack version MUST default to "latest" if not provided.

