# Buildpack Registry

The Buildpack Registry supports distribution of buildpacks. It provides a centralized service that platforms can use to resolve a buildpack ID and version into a concrete buildpackage that can be downloaded and used.

## Table of Contents

- [Buildpack Registry](#buildpack-registry)
  - [Table of Contents](#table-of-contents)
  - [Buildpack ID](#buildpack-id)
  - [Backing Storage](#backing-storage)
    - [Directory Structure](#directory-structure)
    - [File Contents](#file-contents)

## Buildpack ID

Each buildpack in the registry MUST have an ID that adheres to the following restrictions:

* It MUST contain a single `/` character
* The value to the left of the `/`, called the *namespace*, MUST match `[a-z0-9\-\.]{1,253}`
* The value to the right of the `/`, called the *name*, MUST match `[a-z0-9\-\.]{1,253}`

## Backing Storage

The registry information is stored in a repository called an index. The index is a set of files and directories that contain information about available buildpacks.

The registry index MUST be stored in a Git repository.

The index MUST work across all major Operating Systems.

### Directory Structure

Folders in the index MUST be split by two nested folders. The buildpack ID will determine the name of the folders where:
* The first folder MUST represent the first two characters of the *name* component of the ID
* The second folder MUST represent the third and fourth characters of the *name* component of the ID
* IDs with a name component that is 1-2 characters long MUST be represented by special folders name `1` and `2`.
* IDs with a name component that is 3 characters long MUST be represented by a folder named `3` with a subdirectory using the first two characters of the name

The folders will contain files that represent each buildpack ID where the following is true:
* The `/` character in the ID MUST be replaced with `_`
* The namespace MUST match `[a-z0-9\-\.]{1,253}`
* The name MUST match `[a-z0-9\-\.]{1,253}`

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

### File Contents

Each file in the index MUST represent a buildpack version. The file MUST contain one or more entries representing each version of the buildpack, delimited by a newline. Each version entry:
* MUST be in JSON format
* MUST be minified (stored on a single line)

Multiple entries in the same file MUST NOT have a comma at the end of the line (i.e. there will be no list/array using []). Thus, each index file will contain invalid JSON.

An entry MUST have the following structure:

```
{
  "ns" : "<string>",
  "name": "<string>",
  "version" : "<string>",
  "yanked" : <boolean>,
  "addr" : "<string>",
}
```

The fields are defined as follows:

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
{"ns":"example","name":"java","version":"0.1.0","yanked":false,"addr":"docker.io/cnbs/fake-buildpack@sha256:a9d9038c0cdbb9f3b024aaf4b8ae4f894ea8288ad0c3bf057d1157c74601b906"}
{"ns":"example","name":"java","version":"0.2.0","yanked":false,"addr":"docker.io/cnbs/fake-buildpack@sha256:2560f05307e8de9d830f144d09556e19dd1eb7d928aee900ed02208ae9727e7a"}
{"ns":"example","name":"java","version":"0.2.1","yanked":false,"addr":"docker.io/cnbs/fake-buildpack@sha256:74eb48882e835d8767f62940d453eb96ed2737de3a16573881dcea7dea769df7"}
{"ns":"example","name":"java","version":"0.3.0","yanked":false,"addr":"docker.io/cnbs/fake-buildpack@sha256:8c27fe111c11b722081701dfed3bd55e039b9ce92865473cf4cdfa918071c566"}
```