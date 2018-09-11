# Buildpack Registry API

The following additional elements extend the [`buildpack.toml` data format](../buildpack.md#buildpacktoml-toml) section in the Buildpack Interface Specification.

```toml
[buildpack]

  [publish.Ignore]
  files = ["<files to ignore>"]

  [[publish.Vendor]]
  url = "<url to download>"
  dir = "<directory to download>"
  files = ["<files to keep>"]
```

The list of files to ignore is optional, and will be used by the Buildpack Registry
to prune files from the repository when packaging a buildpack.

The vendor key is optional, and will be used by the Buildpack Registry to download
additional files for inclusion in the buildpack package.
