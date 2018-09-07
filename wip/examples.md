# Buildpack API v3 - Examples

## Go

**/bin/build**

The buildpack provides Go in <cache>/go/.

The buildpack provides Go packages that are dependencies of the app in <cache>/gopath/.

The buildpack provides a compiled binary of the app in <launch>/go-app/bin/.

The buildpack removes Go source code from <app>.

In launch.toml, the buildpack writes a web process type pointed at the compiled binary.

## Ruby

**/bin/build**

The buildpack provides Ruby in <cache>/ruby/.

The buildpack provides gems that are dependencies of the app in <cache>/gems/.

The buildpack provides Ruby in <launch>/ruby/.

The buildpack provides gems that are dependencies of the app in <launch>/gems/.

In <launch>/ruby.toml, the buildpack notes the installed version of ruby.

In <launch>/gems.toml, the buildpack notes a checksum of <app>/Gemfile.lock.

In launch.toml, the buildpack writes a web process type that runs the app.