# Buildpack Registry Specification

This document specifies the interface and storage format of a Buildpack Registry.

A Buildpack Registry is a place to publish, store, and discover buildpacks. A platform MAY resolve a Buildpack ID and version to an artifact stored in a Buildpack Registry.

## Buildpack ID and Version

A buildpack ID MUST be composed of two fields, namespace and name, separated by the `/` character. The ID MUST be defined in the `buildpack.toml` of a buildpack with the following schema:

```
[buildpack]
id = "<namespace>/<name>"
version = "<string>"
```

In addition, a buildpack MUST have a `version` value that is unique to all other buildpacks with the same ID stored in the registry.

## Registry Index

The registry index SHALL be stored in a Git repository such that the index MAY be replicated with a structure that is compatible across all major operating systems.

Files containing index data MUST be store in directories split into two nested folders OR special numerical-named directories. For buildpacks with a name-length equal to or greater than 4 characters, the first folder in the hierarchy MUST be named using the first two characters of the buildpack name, and second folder in the hierarchy MUST be named using third and fourth characters of the name. IDs that are 1-3 characters long MUST be stored in special folders named `1`, `2`, and `3`. When the name is exactly 3 characters long the, a subdir in the `3` will be named using the first 2 characters of the name.

The filename will be the ID, where it matches: `[a-z0-9\-\.]{1,253}`. The `/` in the buildpack ID MUST be replaced by a `_` in the filename.

The following defines the directory and file structure of the index:

```
1
└── <ns>_<name>
2
└── <ns>_<name>
3
└── <name[0:2]>
    └── <ns>_<name>
<name[0:2]>
└── <name[2:4]>
    └── <ns>_<name>
```

Each line of a file representing a buildpack in the buildpack registry index MUST contain JSON formatted data. The file MAY contain multiple entries representing each version of the buildpack split by a newline. Each version entry MUST be minified (stored on a single line), and multiple entries in the same file MUST NOT have a comma at the end . Each index file SHALL contain invalid JSON.

An entry will have the following unminified structure:

```
{
  "ns" : "<string>",
  "name": "<string>",
  "version" : "<string>",
  "yanked" : <boolean>,
  "addr" : "<string>"
}
```

Where:

* `ns` - can represent a set or organization of buildpacks.
* `name` - an identifier that must be unique within a namespace.
* `version` - the version of the buildpack (must match the version in the `buildpack.toml` of the buildpack)
* `yanked` - whether or not the buildpack has been removed from the registry
* `addr` - the address of the image stored in a Docker Registry (ex. `"docker.io/jkutner/lua@sha256:abc123"`). The image reference must use `@digest` and not use `:tag`.


## Example

A buildpack with the ID `example/java` is stored in the registry under this file structure:

```
ja
└── va
    └── example_java
```

The `example_java` file contains the following data:

```
{"ns":"example","name":"java","version":"0.1.0","yanked":false,"addr":"docker.io/cnb-example/java-buildpack@sha256:a9d9038c0cdbb9f3b024aaf4b8ae4f894ea8288ad0c3bf057d1157c74601b906"}
{"ns":"example","name":"java","version":"0.2.0","yanked":false,"addr":"docker.io/cnb-example/java-buildpack@sha256:2560f05307e8de9d830f144d09556e19dd1eb7d928aee900ed02208ae9727e7a"}
{"ns":"example","name":"java","version":"0.2.1","yanked":false,"addr":"docker.io/cnb-example/java-buildpack@sha256:74eb48882e835d8767f62940d453eb96ed2737de3a16573881dcea7dea769df7"}
{"ns":"example","name":"java","version":"0.3.0","yanked":false,"addr":"docker.io/cnb-example/java-buildpack@sha256:8c27fe111c11b722081701dfed3bd55e039b9ce92865473cf4cdfa918071c566"}
```



