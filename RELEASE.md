# Release Process

## Scaffolding a New Release

When planning a spec release a maintainer should:
1. Create a milestone for the release (e.g. [`Buildpack 0.5`](https://github.com/buildpacks/spec/milestone/9))
1. Create a release branch from `main` branch with name `<spec>/<version>` (e.g. `buildpack/0.5`) or `extensions/<spec>/version>` for extension specifications.
1. Update the version of the relevant spec in `README.md` and the specification file (e.g. see commit [791363e](https://github.com/buildpacks/spec/commit/791363e329d22a7a116ac09df6e8c739ef21383e)).
1. Push the branch to the repo.


## Release Planning

Issues that are scheduled for a given release should be added to the milestone. The core team will regularly review and adjust the contents of upcoming milestones at the weekly [core team sync](https://github.com/buildpacks/community#core-team).

## Accepting Contributions to a Release
When a PR is opened, the first core team member to review should:
1. Convert to draft if the PR implements an issue that has not been scheduled in a milestone.
1. Ensure the PR is pointed at the correct branch.
1. Request changes if the PR modifies any of the other specifications.
1. Add the PR to a milestone, if this was not done already.
1. Add the matching `api/*` label if this was not done already (e.g. [api/buildpack](https://github.com/buildpacks/spec/pulls?q=is%3Apr+label%3Aapi%2Fbuildpack+is%3Aopen) )

Spec changes must be approved by a super-majority of the core team.

## Finalizing the Release
When all issues and PRs in a given milestone are complete, a core team member may:
1. Create a PR to "finalize" the release, merge the release branch to `main`.
1. Create a draft release and write release notes.

## Approving the Final Release
All core team members and any interested maintainers should:
1. Review the release in its entirety and provide last minute feedback as needed.
1. Review the release notes if they desire.
1. Approve the final PR.

## Cutting the Release
Once the final PR has the required approvals, the person that opened it should:
1. Merge the PR.
1. Publish the release.
1. Close the milestone.
1. Delete the release branch.
