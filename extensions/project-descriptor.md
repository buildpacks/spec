# Project Descriptor

A project descriptor is a file that MAY contain configuration for apps, services, functions, and buildpacks. By default, the file SHOULD be named `project.toml` and located in the root directory of a project's repository. A platform SHOULD read the file to enrich the build process.

## Table of Contents

<!-- Using https://github.com/yzhang-gh/vscode-markdown to manage toc -->
- [Project Descriptor](#project-descriptor)
  - [Table of Contents](#table-of-contents)
  - [Schema](#schema)
  - [`[project]`](#project)
    - [`[[project.licenses]]`](#projectlicenses)
  - [`[build]`](#build)
    - [`[build.include]` and `[build.exclude]`](#buildinclude-and-buildexclude)
    - [`[[build.buildpacks]]`](#buildbuildpacks)
    - [`[[build.env]]`](#buildenv)
  - [`[metadata]`](#metadata)
  - [Example](#example)

## Schema

The TOML schema of the project descriptor is the following:

```toml
[project]
id = "<string>" # machine readable
name = "<string>" # human readable
version = "<string>"
authors = ["<string>"]
documentation-url = "<url>"
source-url = "<url>"

[[project.licenses]]
type = "<string>"
uri = "<uri>"

[build]
builder = "<string>"
include = ["<string>"]
exclude = ["<string>"]
[[build.buildpacks]]
id = "<string>"
version = "<string>"
uri = "<string>"
[[build.env]]
name = "<string>"
value = "<string>"
[metadata]
# additional arbitrary keys allowed
```

The following sections describe each part of the schema in detail.

## `[project]`

The top-level `[project]` table MAY contain configuration about the repository, including `id` and `version`. It MAY also include metadata about how it is authored, documented, and version controlled.

The `project.id`

```toml
[project]
id = "<string>"
name = "<string>"
version = "<string>"
```

* `id` - (optional) the machine readable identifier of the project (ex. "com.example.myservice")
* `name` - (optional) the human readable name of the project (ex. "My Example Service")
* `version` - (optional) and arbitrary string representing the version of the project
* `authors` - (optional) the names and/or email addresses of the project's authors
* `documentation-url` - (optional) a URL to the documentation for the project
* `source-url` - (optional) a URL to the source code for the project

### `[[project.licenses]]`

An optional list of project licenses.

* `type` - This MAY use the [SPDX 2.1 license expression](https://spdx.org/spdx-specification-21-web-version), but is not limited to identifiers in the [SPDX Licenses List](https://spdx.org/licenses/).
* `uri` - If this project is using a nonstandard license, then this key MAY be specified in lieu of or in addition to `type` to point to the license.

## `[build]`

The top-level `[build]` table MAY contain configuration about how to build the project. It MAY include the following keys and others defined below in their individual sub-sections - 

* `builder` - (optional) the builder image to use (ex. "cnbs/sample-builder:bionic")

### `[build.include]` and `[build.exclude]`

An optional list of files to include in the build (while excluding everything else):

```toml
[build]
include = [
    "cmd/",
    "go.mod",
    "go.sum",
    "*.go"
]
```

A list of files to exclude from the build (while including everything else):

```toml
[build]
exclude = [
    "spec/"
]
```

The `.gitignore` pattern is used in both cases. The `exclude` and `include` keys are mutually exclusive, and if both are present the Lifecycle will error out.

Any files that are excluded (either via `include` or `exclude`) MUST BE excluded before the build (i.e. not only excluded from the final image).

If both `exclude` and `include` are defined, the build process MUST result in an error.

### `[[build.buildpacks]]`

The build table MAY contain an array of buildpacks. The schema for this table is:

```toml
[[build.buildpacks]]
id = "<buildpack ID (optional)>"
version = "<buildpack version (optional default=latest)>"
uri = "<url or path to the buildpack (optional default=urn:buildpack:<id>)"
```

This defines the buildpacks that a platform should use on the repo.

Either an `id` or a `uri` MUST be included, but MUST NOT include both. If `uri` is provided, `version` MUST NOT be allowed.

### `[[build.env]]`

Used to set environment variables at build time, for example:

```toml
[[build.env]]
name = "JAVA_OPTS"
value = "-Xmx1g"
```

## `[metadata]`

This table includes a some defined keys, but additional keys are not validated. It can be used to add platform specific metadata. For example:

```toml
[metadata.heroku]
pipeline = "foobar"
```

## Example

```toml
[project]
id = "io.buildpacks.my-app"
version = "0.1"

[build]
builder = "cnbs/sample-builder:bionic"
include = [
    "cmd/",
    "go.mod",
    "go.sum",
    "*.go"
]

[[build.buildpacks]]
id = "io.buildpacks/java"
version = "1.0"

[[build.buildpacks]]
id = "io.buildpacks/nodejs"
version = "1.0"

[metadata]
foo = "bar"

[metadata.fizz]
buzz = ["a", "b", "c"]
```
