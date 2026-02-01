# AutoPub Action

A GitHub Action for automatic package releases using [autopub](https://github.com/autopub/autopub). Automatically publish your Python packages to PyPI when pull requests are merged.

## Features

- **Simple interface**: Single action with `check`, `prepare`, `build`, and `publish` commands
- **Automatic artifact management**: Seamlessly passes release info between jobs
- **GitHub integration**: PR comments and GitHub releases out of the box
- **Flexible publishing**: Supports both PyPI trusted publishing (OIDC) and token-based authentication
- **Version flexibility**: Use latest stable, pre-release, or pin to a specific version

## Quick Start

### 1. Add a RELEASE.md to your PR

```markdown
---
release type: patch
---

Fixed a bug in the authentication module that caused login failures.
```

Release type can be `major`, `minor`, or `patch` following [semver](https://semver.org/).

### 2. Set up workflows

**PR Check** (`.github/workflows/pr.yml`):

```yaml
name: PR Check

on:
  pull_request_target:
    branches: [main]

permissions:
  contents: read
  pull-requests: write

jobs:
  check-release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          ref: "refs/pull/${{ github.event.number }}/merge"
          # Only checkout the files needed for the check - faster and more secure
          sparse-checkout: |
            RELEASE.md
            pyproject.toml
          sparse-checkout-cone-mode: false

      - uses: autopub/autopub-action@v1
        with:
          command: check
          github-token: ${{ secrets.GITHUB_TOKEN }}
```

**Release** (`.github/workflows/release.yml`):

```yaml
name: Release

on:
  push:
    branches: [main]

permissions:
  contents: write
  id-token: write  # Required for trusted publishing

jobs:
  check:
    runs-on: ubuntu-latest
    outputs:
      has-release: ${{ steps.check.outputs.has-release }}
    steps:
      - uses: actions/checkout@v4

      - uses: autopub/autopub-action@v1
        id: check
        with:
          command: check
          github-token: ${{ secrets.GITHUB_TOKEN }}

  release:
    needs: check
    if: needs.check.outputs.has-release == 'true'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          token: ${{ secrets.PAT_TOKEN }}

      - uses: autopub/autopub-action@v1
        with:
          command: prepare

      - uses: autopub/autopub-action@v1
        with:
          command: build

      - uses: autopub/autopub-action@v1
        with:
          command: publish
          github-token: ${{ secrets.PAT_TOKEN }}
```

## Inputs

| Input | Required | Default | Description |
| ----- | -------- | ------- | ----------- |
| `command` | Yes | - | Command to run: `check`, `prepare`, `build`, `publish` |
| `github-token` | No | `${{ github.token }}` | GitHub token for PR comments and releases |
| `pypi-token` | No | - | PyPI token (alternative to trusted publishing) |
| `autopub-version` | No | `latest` | Autopub version: `latest`, `pre-release`, or specific version |
| `git-username` | No | `autopub` | Git username for release commits |
| `git-email` | No | `autopub@autopub` | Git email for release commits |
| `extra-plugins` | No | - | Additional plugins to load (comma-separated) |
| `upload-artifact` | No | `true` | Auto-upload `.autopub` artifact after check |
| `download-artifact` | No | `true` | Auto-download `.autopub` artifact before other commands |
| `artifact-name` | No | `autopub-data` | Name for the artifact |
| `publish-repository` | No | - | Custom repository URL for publishing |
| `fail-on-missing` | No | `false` | Fail the action if no valid RELEASE.md found during check |

## Outputs

| Output | Description |
| ------ | ----------- |
| `has-release` | `true` if a valid `RELEASE.md` was found |
| `version` | The computed release version (available after `prepare`) |
| `release-type` | The release type: `major`, `minor`, or `patch` |
| `release-notes` | The release notes from `RELEASE.md` |

## Commands

### `check`

Validates the `RELEASE.md` file and prepares release information.

- Parses the release file and validates the format
- Creates `.autopub/release_info.json` with release metadata
- Posts a comment on the PR with the changelog preview (if GitHub plugin enabled)
- Uploads the `.autopub` artifact for use by subsequent jobs (only when a valid release is found and `upload-artifact` is enabled)

```yaml
- uses: autopub/autopub-action@v1
  with:
    command: check
    github-token: ${{ secrets.GITHUB_TOKEN }}
```

### `prepare`

Bumps the version and updates the changelog.

- Downloads the `.autopub` artifact from the check step
- Updates version in `pyproject.toml` and `__init__.py`
- Updates `CHANGELOG.md` with the release notes

```yaml
- uses: autopub/autopub-action@v1
  with:
    command: prepare
```

### `build`

Builds the package using the configured package manager (uv, poetry, or pdm).

```yaml
- uses: autopub/autopub-action@v1
  with:
    command: build
```

### `publish`

Publishes the package to PyPI and creates a GitHub release.

- Publishes to PyPI using trusted publishing or token
- Creates a git tag and pushes changes
- Creates a GitHub release with the changelog
- Posts a comment on the PR confirming the release

```yaml
- uses: autopub/autopub-action@v1
  with:
    command: publish
    github-token: ${{ secrets.BOT_TOKEN }}
```

## Authentication

### GitHub Token

For basic PR check functionality, the default `GITHUB_TOKEN` works:

```yaml
github-token: ${{ secrets.GITHUB_TOKEN }}
```

For pushing commits and creating releases, you'll need a Personal Access Token (PAT) or a GitHub App token with write permissions:

```yaml
github-token: ${{ secrets.PAT_TOKEN }}
```

### PyPI Publishing

#### Trusted Publishing (Recommended)

PyPI supports [trusted publishing](https://docs.pypi.org/trusted-publishers/) using OpenID Connect. This is the most secure option as it doesn't require storing tokens.

1. Configure your PyPI project to trust your GitHub repository
2. Add the `id-token: write` permission to your workflow:

```yaml
permissions:
  id-token: write
```

The action will automatically use trusted publishing when no `pypi-token` is provided.

#### Token-based Publishing

Alternatively, use a PyPI token:

```yaml
- uses: autopub/autopub-action@v1
  with:
    command: publish
    pypi-token: ${{ secrets.PYPI_TOKEN }}
```

## Configuration

Configure autopub in your `pyproject.toml`:

```toml
[tool.autopub]
project-name = "My Project"
git-username = "release-bot"
git-email = "bot@example.com"

# Enable additional plugins
plugins = ["github"]

# Plugin-specific configuration
[tool.autopub.plugins.github]
include_sponsors = true
create_discussions = true
```

## Advanced Usage

### Using Pre-release Versions

To test the latest features before they're released:

```yaml
- uses: autopub/autopub-action@v1
  with:
    command: check
    autopub-version: pre-release
```

### Pinning Autopub Version

For reproducible builds:

```yaml
- uses: autopub/autopub-action@v1
  with:
    command: check
    autopub-version: "1.0.0"
```

### Manual Artifact Management

If you need custom artifact handling:

```yaml
- uses: autopub/autopub-action@v1
  with:
    command: check
    upload-artifact: false

- uses: actions/upload-artifact@v4
  with:
    name: my-custom-artifact
    path: .autopub
```

### Custom Git Identity

```yaml
- uses: autopub/autopub-action@v1
  with:
    command: publish
    git-username: "My Bot"
    git-email: "bot@mycompany.com"
```

### Publishing to Custom Repository

```yaml
- uses: autopub/autopub-action@v1
  with:
    command: publish
    publish-repository: "https://my-pypi-server.com/simple/"
```

### Requiring RELEASE.md in PRs

To enforce that all PRs include a `RELEASE.md` file:

```yaml
- uses: autopub/autopub-action@v1
  with:
    command: check
    fail-on-missing: "true"
```

This will fail the workflow if no valid `RELEASE.md` is found, useful for PR checks.

## Troubleshooting

### "No valid RELEASE.md found"

The `check` command returns `has-release: false` when:

- `RELEASE.md` doesn't exist
- The file format is invalid
- The release type is not `major`, `minor`, or `patch`

### Artifact not found

If `prepare`, `build`, or `publish` fail with artifact errors:

1. Ensure `check` completed successfully in a previous job
2. Verify the `artifact-name` matches between jobs
3. Check that `upload-artifact: true` (default) was used in the check step

### Permission denied pushing to repository

Ensure your token has write permissions:

- Use a PAT with `contents: write` scope
- Or use a GitHub App installation token
- Don't use the default `GITHUB_TOKEN` for pushing (it has read-only access to the repository in `pull_request` events)

## License

AGPL-3.0 - see [LICENSE](LICENSE) for details.
