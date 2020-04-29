# Buildpack Registry

The Buildpack Registry supports distribution of buildpacks. It provides a centralized service that platforms can use to resolve a buildpack ID and version into a concrete buildpackage that can be downloaded and used.

# Buildpack ID

Each buildpack in the registry MUST have an ID that adheres to the following restrictions:

* It MUST contain a single `/` character
* The value to the left of the `/`, called the namespace, MUST match `[a-z0-9\-\.]{1,253}`
* The value to the right of the `/`, called the name, MUST match `[a-z0-9\-\.]{1,253}`

# Backing Storage

The registry information is stored in a repository called an index. The index is a set of files and directories that contain information about available buildpacks.

The registry index MUST be stored in a Git repository.

The index MUST work across all major Operating Systems.

## Directory Structure

Folders in the index MUST be split by two nested folders. The buildpack ID will determine the name of the folders where:
* The first folder MUST represent the first two characters
* The second folder MUST represent the third and fourth characters
* IDs that are 1-3 characters long MUST be represented by special folders.

The folders will contain files that represent each buildpack ID where the following is true:
* The `/` character in the ID MUST be replaced with `_`
* The namespace MUST match `[a-z0-9\-\.]{1,253}`
* The name MUST match `[a-z0-9\-\.]{1,253}`

Here's an example directory structure:

```
1
├── example_a
└── example_b
2
├── example_aa
└── example_ab
3
├── a
│   ├── foobar_abc
│   └── example_acd
└── b
    ├── example_bcd
    └── example_bed
fo
├── ob
│   ├── samples_fooball
│   └── example_foobar
└── oc
    └── example_foocal
```

The following ids are reserved by Windows, so they aren't allowed as valid ids:

* nul
* con
* prn
* aux
* com1
* com2
* com3
* com4
* com5
* com6
* com7
* com8
* com9
* lpt1
* lpt2
* lpt3
* lpt4
* lpt5
* lpt6
* lpt7
* lpt8
* lpt9

## File Contents

Each file in the index MUST represent a buildpack. The file MUST contain multiple entries representing each version of the buildpack split by a newline. Each version entry:
* MUST be in JSON format
* MUST be minimized (stored on a single line)

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


An example of what this may look like for a single buildpack file:

```
{"ns":"example","name":"ruby","version":"0.1.0","yanked":false,"addr":"docker.io/hone/ruby-buildpack@sha256:a9d9038c0cdbb9f3b024aaf4b8ae4f894ea8288ad0c3bf057d1157c74601b906"}
{"ns":"example","name":"ruby","version":"0.2.0","yanked":false,"addr":"docker.io/hone/ruby-buildpack@sha256:2560f05307e8de9d830f144d09556e19dd1eb7d928aee900ed02208ae9727e7a"}
{"ns":"example","name":"ruby","version":"0.2.1","yanked":false,"addr":"docker.io/hone/ruby-buildpack@sha256:74eb48882e835d8767f62940d453eb96ed2737de3a16573881dcea7dea769df7"}
{"ns":"example","name":"ruby","version":"0.3.0","yanked":false,"addr":"docker.io/hone/ruby-buildpack@sha256:8c27fe111c11b722081701dfed3bd55e039b9ce92865473cf4cdfa918071c566"}
```