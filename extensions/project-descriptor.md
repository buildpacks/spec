# Project Descriptor

A project descriptor is a file that MAY contain configuration for apps, services, functions, and buildpacks. By default, the file SHOULD be named `project.toml` and located in the root directory of a project's repository. A platform SHOULD read the file to enrich the build process.

## Table of Contents

<!-- Using https://github.com/yzhang-gh/vscode-markdown to manage toc -->
- [Project Descriptor](#project-descriptor)
  - [Table of Contents](#table-of-contents)
  - [Schema Version](#schema-version)
  - [Special Value Types](#special-value-types)
  - [Top Level Tables](#top-level-tables)
    - [Non-`_` Tables](#non-_-tables)
    - [`_`](#_)
      - [`_.licenses` (optional)](#_licenses-optional)
      - [`_.metadata` (optional)](#_metadata-optional)
    - [`io.buildpacks` (optional)](#iobuildpacks-optional)
      - [`io.buildpacks.builder` (optional)](#iobuildpacksbuilder-optional)
      - [`io.buildpacks.include` (optional) and `io.buildpacks.exclude` (optional)](#iobuildpacksinclude-optional-and-iobuildpacksexclude-optional)
      - [`io.buildpacks.group` (optional)](#iobuildpacksgroup-optional)
      - [`io.buildpacks.pre.group` (optional)](#iobuildpackspregroup-optional)
      - [`io.buildpacks.post.group` (optional)](#iobuildpackspostgroup-optional)
      - [`io.buildpacks.env.build` (optional)](#iobuildpacksenvbuild-optional)
  - [Example](#example)

## Schema Version

This document specifies Project Descriptor Schema Version `0.2`.

The Schema Version format follows the form of the [Buildpack API Version](https://github.com/buildpacks/spec/blob/main/buildpack.md#buildpack-api-version):

* MUST be in form <major>.<minor> or <major>, where <major> is equivalent to <major>.0
* When <major> is greater than 0 increments to <minor> SHALL exclusively indicate additive changes

## Special Value Types

* `schema-version` - A string that follows the format of [Buildpack API Version](https://github.com/buildpacks/spec/blob/main/buildpack.md#buildpack-api-version).
* `uri` - A string that follows the format of [RFC3986](https://tools.ietf.org/html/rfc3986).

## Top Level Tables

### Non-`_` Tables

All other tables besides `_` will use reverse domains, i.e. buildpacks.io will be `[io.buildpacks]`. These tables can be optionally versioned with a schema version number using the `schema-version` field. All these tables are optional.

### `_`

The TOML schema of the project section of the project descriptor:

```toml
[_]
schema-version = "<schema-version>"
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

The top-level `_` table MAY contain configuration about the repository, including `id` and `version`. It MAY also include metadata about how it is authored, documented, and version controlled. It MUST contain `schema-version`  to denote which schema version the descriptor is using.

```toml
[_]
schema-version = "<string>"
id = "<string>"
name = "<string>"
version = "<string>"
authors = ["<string>"]
documentation-url = "<uri>"
source-url = "<uri>"
```

* `schema-version` - version identifier for the schema of the `_` table and structure of the project descriptor file.
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

This is the Cloud Native Buildpacks' section of the project descriptor. The TOML schema is the following:

```
[io.buildpacks]
builder = "<string>"
include = ["<string>"]
exclude = ["<string>"]

[[io.buildpacks.group]]
id = "<string>"
version = "<string>"
uri = "<string>"

  [io.buildpacks.group.script]
  api = "<buildpack api>"
  shell = "<string (optional default=/bin/sh)>"
  inline = "<script contents>"

[[io.buildpacks.build.env]]
name = "<string>"
value = "<string>"
```

#### `io.buildpacks.builder` (optional)

This is the builder image to use (ex. "cnbs/sample-builder:bionic").

#### `io.buildpacks.include` (optional) and `io.buildpacks.exclude` (optional)

An optional list of files to include in the build (while excluding everything else):
This MAY contain a list of files to include in the build (while excluding everything else):

```toml
[io.buildpacks]
include = [
    "cmd/",
    "go.mod",
    "go.sum",
    "*.go"
]
```

A list of files to exclude from the build (while including everything else):

```toml
[io.buildpacks]
exclude = [
    "spec/"
]
```

The `.gitignore` pattern is used in both cases. The `exclude` and `include` keys are mutually exclusive, and if both are present the Lifecycle will error out.

Any files that are excluded (either via `include` or `exclude`) MUST BE excluded before the build (i.e. not only excluded from the final image).

If both `exclude` and `include` are defined, the build process MUST result in an error.

#### `io.buildpacks.group` (optional)

This table MAY contain an array of buildpacks. The schema for this table is:

```toml
[[io.buildpacks.group]]
id = "<buildpack ID (optional)>"
version = "<buildpack version (optional default=latest)>"
uri = "<url or path to the buildpack (optional default=urn:buildpack:<id>)"

  [io.buildpacks.group.script]
  api = "<buildpack api>"
  shell = "<string (optional default=/bin/sh)>"
  inline = "<script contents>"
```

This defines the buildpacks that a platform should use on the repo.

Either a `version`, `uri`, or `script` table MUST be included, but MUST NOT include any combination of these elements.

The `api` and `inline` key MUST be defined in the `script` table. The value of the `inline` key will be used as the build script for the [inline buildpack](#Definitions) this entry represents. The value of the `api` key defines its Buildpack API compatibility, and the `shell` key defines the shell used to execute the `inline` script.

#### `io.buildpacks.pre.group` (optional)

This table MAY contain a list of buildpacks to insert at the beginning of an automatically detected group. Given an order with multiple groups, the list of `pre` buildpacks will be inserted at the beginning of each automatically detected group such that they are run as if they were originally included in the group. Each phase of the injected buildpack(s) will execute as normal.
[[build.post.buildpacks]]

The schema for this table is identical to [`io.buildpacks.group`](#iobuildpacksgroup-optional)

#### `io.buildpacks.post.group` (optional)

This table MAY contain a list of buildpacks to insert at the end of an automatically detected group. Given an order with multiple groups, the list of `post` buildpacks will be inserted at the end of each group such that they are run as if they were originally included in the group. Each phase of the injected buildpack(s) will execute as normal.

The schema for this table is identical to [`io.buildpacks.group`](#iobuildpacksgroup-optional)

#### `io.buildpacks.env.build` (optional)

This table MAY be used to set environment variables at build time, for example:

```toml
[[io.buildpacks.env.build]]
name = "JAVA_OPTS"
value = "-Xmx1g"
```

## Terminology

* **Inline Buildpack** - a type of buildpack that can be defined in the same repo as the app it is used with

## Example

```toml
[_]
id = "io.buildpacks.my-app"
version = "0.1"

[_.metadata]
cdn = "https://cdn.example.com"

[[_.metadata.assets]]
url = "https://cdn.example.com/assets/foo.jar"
checksum = "3b1b39893d8e34a6d0bd44095afcd5c4"

buzz = ["a", "b", "c"]

[io.buildpacks]
builder = "cnbs/sample-builder:bionic"
include = [
    "cmd/",
    "go.mod",
    "go.sum",
    "*.go"
]

[[io.buildpacks.group]]
id = "io.buildpacks/java"
version = "1.0"

[[io.buildpacks.group]]
id = "io.buildpacks/nodejs"
version = "1.0"
```
