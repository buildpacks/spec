## Contributing to a Release

Contributions should generally adhere to the following guidelines:

1. Contributions should always be made to a release branch and never to the `main` branch, except when modifying [`README.md`](README.md) or files which are not formally part of the specification such as this one ([`RELEASE.md`](RELEASE.md)).
1. Other than the first commit (which bumps the specification version in `README.md`), all PRs to a given release branch should only modify the named specification (e.g. PRS to `buildpack/0.5` should exclusively modify [`buildpack.md`](buildpack.md)). If a contributor wishes to modify two specification files (for example to move contend from `buildpack.md` to `platform.md`) two PRs should be opened against the release branches for the respective specifications.
1. A PR should either:
   1. Implement an issue that has been scheduled in an upcoming milestone, in which case it should point at the branch that matches the milestone.
   1. Make a non-functional typographical or organizational improvement, in which case it should point at the branch for the next release of the given specification.
1. Please do not open a PR implementing an issue that is not yet associated milestone. Instead, please comment on the issue and work with the core team to get it scheduled.
1. Please do not propose major changes to the specification exclusively via a PR to this repo. Instead, please propose an [RFC](https://github.com/buildpacks/rfcs) or provide feedback via an issue so that it may eventually be addressed in an RFC. However, a draft PR may accompany/clarify an RFC.

See [`RELEASE.md`](./RELEASE.md) for more details on the release process.
