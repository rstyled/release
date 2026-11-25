# release

One release setup for every rstyled repository: Conventional Commits decide the next version, [knope] works it out,
each release is a Git tag `vX.Y.Z` with a GitHub release, and a repository that publishes a Docker image tags the
image with the same version. Nothing is published to a package registry, and a release never adds a commit: the tag is
the version.

[knope]: https://knope.tech

## What a repository gets

On every push to the default branch:

1. **Version.** knope reads the commits since the last `vX.Y.Z` tag. A `feat`, `fix`, `perf` or breaking commit makes
   this commit a release; anything else (docs, chore, ci, ...) leaves the version where it is.
2. **Image** (repositories that publish one). The image is built and pushed as `:<commit>` and `:latest`, as before,
   and also as `:X.Y.Z` when the commit is a release. Every image carries the labels
   `org.opencontainers.image.version` (`X.Y.Z`, or `X.Y.Z-N-gSHA` between releases), `.revision` and `.source`.
3. **Release.** Once the image is published, the commit is tagged `vX.Y.Z` and gets a GitHub release listing its
   features, fixes and breaking changes, and naming the image by digest.

On a pull request, the image is built but not published, and the title and every commit must be Conventional Commits.

| Since the last release | 1.0 and later | 0.x |
| --- | --- | --- |
| `feat!:`, `fix!:`, ... or a `BREAKING CHANGE:` footer | major | minor |
| `feat:` | minor | patch |
| `fix:`, `perf:` | patch | patch |
| `refactor:`, `test:`, `docs:`, `build:`, `ci:`, `chore:`, `revert:`, `style:` | no release | no release |

A repository with no `vX.Y.Z` tag makes its first release, `0.0.1` unless `initial-version` says otherwise, on its
next push. To leave 0.x, release `1.0.0` by hand once: `gh release create v1.0.0 --target <commit>`.

## Adopting it

Grant the calling workflow the permissions its jobs need; the jobs ask for no more than these.

A repository that publishes an image (`.github/workflows/publish.yml`):

```yaml
name: Publish
on:
  pull_request:
    paths: ['publish/**', '.github/workflows/publish.yml']
  push:
    branches: [main]
  workflow_dispatch:
permissions:
  contents: write
  packages: write
concurrency:
  group: publish-${{ github.ref }}
  cancel-in-progress: ${{ github.event_name == 'pull_request' }}
jobs:
  publish:
    uses: rstyled/release/.github/workflows/image.yml@v0.0
    with:
      dockerfile: publish/Dockerfile
      runs-on: warp-ubuntu-latest-x64-4x
```

A repository with nothing to publish (`.github/workflows/release.yml`):

```yaml
name: Release
on:
  push:
    branches: [main]
permissions:
  contents: write
concurrency:
  group: release-${{ github.ref }}
jobs:
  release:
    uses: rstyled/release/.github/workflows/release.yml@v0.0
```

Every repository (`.github/workflows/commits.yml`):

```yaml
name: Conventional commits
on:
  pull_request:
    types: [opened, edited, synchronize, reopened]
permissions:
  contents: read
jobs:
  commits:
    uses: rstyled/release/.github/workflows/commits.yml@v0.0
```

A repository with its own build and deploy (rstyle-mcp-server) uses the actions inside its workflow instead:
`rstyled/release/version@v0.0` early, so the build can carry the version, and `rstyled/release/release@v0.0` last, once
the deploy succeeds. Both need a checkout with `fetch-depth: 0`.

| Action | Inputs | Outputs |
| --- | --- | --- |
| `version` | `initial-version` | `version`, `tag`, `release`, `tagged`, `describe` |
| `release` | `version`, `notes`, `token` | `released` |
| `commits` | `title`, `base`, `head` (from the pull request by default) | |

The knope config is this repository's (`lib/knope` writes it for each run), so a repository needs no `knope.toml`, and
`lib/knope` refuses to run with one. To run knope locally: `KNOPE=$(which knope) path/to/release/lib/knope release --dry-run`.

## Commits

`<type>(<optional scope>): <description>`, with a lowercase scope: `fix(cpg): keep a loop's bounds`. A squash merge
makes the pull request's title the commit, and a merge keeps each commit, so the check holds both to the format.
git's own `Revert "..."` subject passes and releases nothing.

The private repositories are on GitHub's free plan, which has no branch protection, so a failing check can't block a
merge; it's on whoever merges to see it. To be stopped at commit time, install the hook in each checkout:

```sh
curl -fsSL https://raw.githubusercontent.com/rstyled/release/v0.0/hooks/commit-msg \
  -o "$(git rev-parse --git-path hooks)/commit-msg" && chmod +x "$(git rev-parse --git-path hooks)/commit-msg"
```

## Versioning this repository

It releases itself the same way, from `0.0.1`, and moves a line tag to each release: before 1.0, `v0.MINOR` (`v0.0`
for 0.0.x), since a breaking change there bumps the minor version; from 1.0, `vMAJOR`. Callers pin the line,
`@v0.0` today, so they take fixes and features and keep their interface. A breaking change to an input, output or
behavior is a `feat!:` and starts the next line (`v0.1`), which callers move to by changing their pin. `tests/run` covers the scripts; set `KNOPE` to a knope 0.23.2 binary to run it offline.

The repository is public because a public repository (rstyle) can't call a workflow in a private one. It holds no
secrets: everything runs with the calling repository's `GITHUB_TOKEN`.
