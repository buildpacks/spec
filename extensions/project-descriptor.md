# Project Descriptor

A project descriptor is a file that MAY contain configuration for apps, services, functions, and buildpacks. By default, the file SHOULD be named `project.toml` and located in the root directory of a project's repository. A platform SHOULD read the file to enrich the build process.

## Table of Contents

<!-- Using https://github.com/yzhang-gh/vscode-markdown to manage toc -->
- [Project Descriptor](#project-descriptor)
  - [Table of Contents](#table-of-contents)
  - [API Version](#api-version)
  - [Special Value Types](#special-value-types)
  - [Top Level Tables](#top-level-tables)
    - [Non-`_` Tables](#non-_-tables)
    - [`_`](#_)
      - [`_.licenses` (optional)](#_licenses-optional)
      - [`_.metadata` (optional)](#_metadata-optional)
    - [`io.buildpacks` (optional)](#iobuildpacks-optional)
      - [`io.buildpacks.build.builder` (optional)](#iobuildpacksbuildbuilder-optional)
      - [`io.buildpacks.build.include` (optional) and `io.buildpacks.build.exclude` (optional)](#iobuildpacksbuildinclude-optional-and-iobuildpacksbuildexclude-optional)
      - [`io.buildpacks.build.buildpacks` (optional)](#iobuildpacksbuildbuildpacks-optional)
      - [`io.buildpacks.build.env` (optional)](#iobuildpacksbuildenv-optional)
  - [Example](#example)

## API Version

This document specifies Project Descriptor API Version `0.2`.

The API version format follows the form of [Buildpack API Version](https://github.com/buildpacks/spec/blob/main/buildpack.md#buildpack-api-version).

## Special Value Types

* `api` - A string that follows the format of [Buildpack API Version](https://github.com/buildpacks/spec/blob/main/buildpack.md#buildpack-api-version).
* `uri` - A string that follows the format of [RFC3986](https://tools.ietf.org/html/rfc3986).

## Top Level Tables

### Non-`_` Tables

All other tables besides `_` will use reverse domains, i.e. buildpacks.io will be `[io.buildpacks]`. These tables can be optionally versioned with a schema version API number using the `api` field. All these tables are optional.

### `_`

The TOML schema of the project section of the project descriptor:

```toml
[_]
api = "<api>"
id = "<string>" # machine readable
name = "<string>" # human readable
version = "<string>"
authors = ["<string>"]
documentation-url = "<uri>"
source-url = "<uri>"

[[_.licenses]]
type = "<string>"
uri = "<uri>"

[_.metadata]
# additional arbitrary keys allowed
```

The top-level `_` table MAY contain configuration about the repository, including `id` and `version`. It MAY also include metadata about how it is authored, documented, and version controlled. It MUST contain `api`  to denote which version of this API the descriptor is using.

```toml
[_]
api = "<string>"
id = "<string>"
name = "<string>"
version = "<string>"
authors = ["<string>"]
documentation-url = "<uri>"
source-url = "<uri>"
```

* `api` - version identifier for the schema of the `_` table and structure of the project descriptor file. This version format follows the rules of the [Buildpack API Version](https://github.com/buildpacks/spec/blob/main/buildpack.md#buildpack-api-version).
* `id` - (optional) the machine readable identifier of the project (ex. "com.example.myservice")
* `name` - (optional) the human readable name of the project (ex. "My Example Service")
* `version` - (optional) and arbitrary string representing the version of the project
* `authors` - (optional) the names and/or email addresses of the project's authors
* `documentation-url` - (optional) a URL to the documentation for the project.
* `source-url` - (optional) a URL to the source code for the project

#### `_.licenses` (optional)

This table MAY contain project licenses.

```toml
[[_.licenses]]
type = "<string>"
uri = "<uri>"
```

* `type` - This MAY use the [SPDX 2.1 license expression](https://spdx.org/spdx-specification-21-web-version), but is not limited to identifiers in the [SPDX Licenses List](https://spdx.org/licenses/).
* `uri` - If this project is using a nonstandard license, then this key MAY be specified in lieu of or in addition to `type` to point to the license.

#### `_.metadata` (optional)

This is a free form table for users to use as they see fit. The keys in this table are not validated.

```toml
[_.metadata.foo]
checksum = "a28a0d7772df1f918da2b1102da4ff35"
```


### `io.buildpacks` (optional)

This is the Cloud Native Buildpacks' section of the project descriptor. It contains a different API version from [`_`](#_). The TOML schema is the following:

```
[io.buildpacks]

[io.buildpacks.build]
builder = "<string>"
include = ["<string>"]
exclude = ["<string>"]

[[io.buildpacks.build.buildpacks]]
id = "<string>"
version = "<string>"
uri = "<string>"

[[io.buildpacks.build.env]]
name = "<string>"
value = "<string>"
```

* `api` - version identifier for the schema of the `io.buildpacks` table.

#### `io.buildpacks.build.builder` (optional)

This is the builder image to use (ex. "cnbs/sample-builder:bionic").

#### `io.buildpacks.build.include` (optional) and `io.buildpacks.build.exclude` (optional)

An optional list of files to include in the build (while excluding everything else):
This MAY contain a list of files to include in the build (while excluding everything else):

```toml
[io.buildpacks.build]
include = [
    "cmd/",
    "go.mod",
    "go.sum",
    "*.go"
]
```

A list of files to exclude from the build (while including everything else):

```toml
[io.buildpacks.build]
exclude = [
    "spec/"
]
```

The `.gitignore` pattern is used in both cases. The `exclude` and `include` keys are mutually exclusive, and if both are present the Lifecycle will error out.

Any files that are excluded (either via `include` or `exclude`) MUST BE excluded before the build (i.e. not only excluded from the final image).

If both `exclude` and `include` are defined, the build process MUST result in an error.

#### `io.buildpacks.build.buildpacks` (optional)

This table MAY contain an array of buildpacks. The schema for this table is:

```toml
[[io.buildpacks.build.buildpacks]]
id = "<buildpack ID (optional)>"
version = "<buildpack version (optional default=latest)>"
uri = "<url or path to the buildpack (optional default=urn:buildpack:<id>)"
```

This defines the buildpacks that a platform should use on the repo.

Either an `id` or a `uri` MUST be included, but MUST NOT include both. If `uri` is provided, `version` MUST NOT be allowed.

#### `io.buildpacks.build.env` (optional)

This table MAY be used to set environment variables at build time, for example:

```toml
[[io.buildpacks.build.env]]
name = "JAVA_OPTS"
value = "-Xmx1g"
```

## Example

```toml
[_]
id = "io.buildpacks.my-app"
version = "0.1"

[_.metadata]
foo = "bar"

[_.metadata.fizz]
buzz = ["a", "b", "c"]

[io.buildpacks.build]
builder = "cnbs/sample-builder:bionic"
include = [
    "cmd/",
    "go.mod",
    "go.sum",
    "*.go"
]

[[io.buildpacks.build.buildpacks]]
id = "io.buildpacks/java"
version = "1.0"

[[io.buildpacks.build.buildpacks]]
id = "io.buildpacks/nodejs"
version = "1.0"
```