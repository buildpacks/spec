# Project Descriptor

A project descriptor is a file that contains configuration for apps, services, functions, and buildpacks. By default, the file is named `project.toml` and located in the root directory of a project's repository.

## Schema

The TOML schema of the project descriptor is the following:

```toml
[project]
id = "<string>" # machine readble
name = "<string>" # human readable
version = "<string>"
authors = ["<string>"]
documentation-url = "<url>"
source-url = "<url>"

[[project.licenses]]
type = "<string>"
uri = "<uri>"

[[build.buildpacks]]
id = "<string>"
version = "<string>"
uri = "<string>"

[[build.env]]
name = "<string>"
value = "<string>"
```

The following sections describe each part of the schema in detail.

## `[project]`

The top-level `[project]` table may contain configuration about the repository, including `id` and `version`, but also metadata about how it is authored, documented, and version controlled.

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

## `[[project.licenses]]`

An optional list of project licenses.

* `type` - This may use the [SPDX 2.1 license expression](https://spdx.org/spdx-specification-21-web-version), but is not limited to identifiers in the [SPDX Licenses List](https://spdx.org/licenses/).
* `uri` - If this project is using a nonstandard license, then this key may be specified in lieu of or in addition to `type` to point to the license.

## `[[build.buildpacks]]`

The build table may contain an array of buildpacks. The schema for this table is:

```toml
[[build.buildpacks]]
id = "<buildpack ID (optional)>"
version = "<buildpack version (optional default=latest)>"
uri = "<url or path to the buildpack (optional default=urn:buildpack:<id>)"
```

This defines the buildpacks that a platform should use on the repo.

Either an `id` or a `uri` are required, but not both. If `uri` is provided, `version` is not allowed.

## `[[build.env]]`

Used to set environment variables at build time, for example:

```toml
[[build.env]]
name = "JAVA_OPTS"
value = "-Xmx1g"
```

## Example

```toml
[project]
id = "io.buildpacks.my-app"
version = "0.1"

[[build.buildpacks]]
id = "io.buildpacks/java"
version = "1.0"

[[build.buildpacks]]
id = "io.buildpacks/nodejs"
version = "1.0"

[[build.env]]
name = "JAVA_OPTS"
value = "-Xmx1g"
```
