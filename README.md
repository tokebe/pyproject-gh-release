# Pyproject GitHub Autorelease

Check whether the `pyproject.toml` package version has no release tag yet. If so, parse
`CHANGELOG.md` (assuming [Keep a Changelog](https://keepachangelog.com/en/1.1.0/)
format) and create a new tagged release.

> [!NOTE]
>
> - Only works with statically typed versions in `pyproject.toml`.
> - Requires git tags to be v-prefixed, i.e. `v1.0.0`, and bare versions in
>   `pyproject.toml`

## Usage

```yaml
name: Autorelease

on:
  push:
    branches: [main]

permissions:
  contents: write # Required to make release

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5
        with:
          fetch-depth: 0 # required for tag checking

      - uses: tokebe/pyproject-gh-release@v1
```

## Inputs

| Input             | Default          | Description                                             |
| ----------------- | ---------------- | ------------------------------------------------------- |
| `pyproject`       | `pyproject.toml` | Path to the `pyproject.toml` to read.                   |
| `parse-changelog` | `true`           | Read the matching `CHANGELOG.md` entry for the version. |
| `create-release`  | `true`           | Create the release. Set `false` to run detection only.  |

## Outputs

| Output        | Description                                                       |
| ------------- | ----------------------------------------------------------------- |
| `version`     | Version declared in `pyproject.toml`.                             |
| `is-new`      | `true` if no `v<version>` tag exists yet.                         |
| `changelog`   | `CHANGELOG.md` entry body for this version (empty if not parsed). |
| `release-url` | URL of the created GitHub release (empty if none created).        |

A release is created only when `is-new` is `true`. When `parse-changelog` is `false`,
the release is created with an empty body.

## Requiring maintainer approval

You can require maintainer approval by using this action to detect, and a second job in
a special
[Environment](https://docs.github.com/en/actions/managing-workflow-runs-and-deployments/managing-deployments/managing-environments-for-deployment)
to make the release (so that approval is only requested for new version releases).

To do so:

1. Create an environment (e.g. `release`) in **Settings -> Environments** and add your
   maintainers under **Required reviewers**.
2. Use a second job to make the release:

```yaml
name: Autorelease (with approval)

on:
  push:
    branches: [main]

permissions:
  contents: write

jobs:
  detect:
    runs-on: ubuntu-latest
    outputs:
      is-new: ${{ steps.detect-state.outputs.is-new }}
      version: ${{ steps.detect-state.outputs.version }}
      changelog: ${{ steps.detect-state.outputs.changelog }}
    steps:
      - uses: actions/checkout@v5
        with:
          fetch-depth: 0
      - id: detect-state
        uses: tokebe/pyproject-gh-release@v1
        with:
          create-release: false

  release:
    needs: detect
    # Skip if there isn't a new release to make
    if: needs.detect.outputs.is-new == 'true'
    runs-on: ubuntu-latest
    environment: release # Environment requests approval
    steps:
      - uses: actions/checkout@v5
        with:
          fetch-depth: 0
      - uses: softprops/action-gh-release@v2
        with:
          tag_name: v${{ needs.detect.outputs.version }}
          name: v${{ needs.detect.outputs.version }}
          body: ${{ needs.detect.outputs.changelog }}
```
