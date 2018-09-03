
### buildpack.toml (TOML)

```toml
[[stacks]]
id = "<stack ID, globally unique, ex. io.buildpacks.stack>"
run-contrib = "/path/to/run.Dockerfile"
build-contrib = "/path/to/build.Dockerfile" 
```