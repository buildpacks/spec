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

[build]
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

## `[build.include]` and `[build.exclude]`

A optional list of files to include in the build (while excluding everything else):

```toml
[build]
include = [
    "cmd/",
    "go.mod",
    "go.sum",
    "*.go"
]
```

A list of files to exclude from the build (while including everything else)

```toml
[build]
exclude = [
    "spec/"
]
```

The `.gitignore` pattern is used in both cases. The `exclude` and `include` keys are mutually exclusive, and if both are present the Lifecycle will error out.

Any files that are excluded (either via `include` or `exclude`) will be excluded before the build (i.e. not only exluded from the final image).

If both `exclude` and `include` are defined, the build process will error out.

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

[[build.buildpacks]]
id = "io.buildpacks/java"
version = "1.0"

[[build.buildpacks]]
id = "io.buildpacks/nodejs"
version = "1.0"
```