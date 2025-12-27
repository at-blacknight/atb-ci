# atb-ci - Reusable GitHub Actions Workflows

Reusable GitHub Actions workflows for automated versioning, building, and releasing projects at at-blacknight.

## Features

- **Semantic Versioning**: Automatic version bumping based on conventional commits
- **Multi-Platform Builds**: Support for Go, Docker, npm, and .NET projects
- **Automated Releases**: Create GitHub releases with changelogs automatically
- **Reusable Workflows**: DRY principle with shared stages and build-type workflows

## Quick Start

### Go Projects

Add this workflow to your repository at `.github/workflows/ci.yml`:

```yaml
name: CI
on:
  push:
    branches: [main]
  pull_request:

permissions:
  contents: write

jobs:
  release:
    uses: at-blacknight/atb-ci/.github/workflows/go-release.yml@v1
    with:
      go-version: '1.24'
      binary-name: 'my-app'
      platforms: 'linux/amd64,linux/arm64,darwin/amd64,darwin/arm64,windows/amd64,windows/arm64'
      enable-windows-resource: true
      windows-company-name: 'AT-BlacKnight'
      windows-product-name: 'MyApp'
```

### Conventional Commits

This project uses conventional commits to determine version bumps:

- `feat: add new feature` → Minor version bump (1.2.0 → 1.3.0)
- `fix: bug fix` → Patch version bump (1.2.0 → 1.2.1)
- `feat!: breaking change` → Major version bump (1.2.0 → 2.0.0)
- `docs: documentation` → No release
- `chore: maintenance` → No release

### How It Works

1. **Pull Requests**: Version detection runs in dry-run mode (no release created)
2. **Main Branch**:
   - Detects next version based on commits since last release
   - Builds binaries for all platforms if release needed
   - Creates GitHub release with changelog
   - Commits CHANGELOG.md back to repository

## Available Workflows

### Go Release (`go-release.yml`)

Full Go release pipeline with semantic versioning.

**Inputs:**
- `go-version` - Go version (default: '1.24')
- `binary-name` - Binary name (required)
- `platforms` - Platform list (default: 6 platforms)
- `ldflags-template` - ldflags template (default: '-X main.version={{version}}')
- `enable-windows-resource` - Windows metadata (default: false)
- `windows-company-name` - Company for Windows .exe metadata
- `windows-product-name` - Product for Windows .exe metadata
- `windows-file-description` - Description for Windows .exe metadata
- `windows-copyright` - Copyright for Windows .exe metadata

**What it does:**
1. Detects version using semantic-release (dry-run on PRs)
2. Builds binaries for all platforms (6 by default)
3. Creates GitHub release with changelog
4. Attaches build artifacts (ZIP files + checksums)
5. Commits CHANGELOG.md

### Docker Release (`docker-release.yml`)

Coming soon.

### npm Release (`npm-release.yml`)

Coming soon.

### .NET Release (`dotnet-release.yml`)

Coming soon.

## Architecture

This repository follows a two-tier architecture:

### Tier 1: Shared Stages (`.github/workflows/stages/`)

Reusable components shared between build types:

- **`version-detect.yml`** - Semantic-release dry-run (forecast version)
- **`release-semantic.yml`** - Semantic-release actual (create tag/release)
- **`go-build-matrix.yml`** - Multi-platform Go builds
- More stages for Docker, npm, .NET...

### Tier 2: Build-Type Workflows (`.github/workflows/`)

Orchestrate shared stages for specific build types:

- **`go-release.yml`** - Go: version-detect → go-build → release-semantic
- **`docker-release.yml`** - Docker: version-detect → docker-build → release-semantic
- More workflows for npm, .NET...

**Consumer repos call build-type workflows** (e.g., `go-release.yml`), not individual stages. Stages are internal implementation details.

## Examples

See the `examples/` directory for complete examples:

- `examples/go-cli-app/` - Example Go CLI application (coming soon)
- `examples/docker-app/` - Example Docker application (coming soon)
- `examples/npm-package/` - Example npm package (coming soon)

## Documentation

- [Architecture Overview](docs/architecture.md)
- [Conventional Commits Guide](docs/conventional-commits.md)
- [Go Release Workflow](docs/workflows/go-release.md)

## Development

### Testing Changes

1. Create example repository
2. Reference this repository at `@main` during development
3. Test with real commits and PRs
4. Release tagged version when stable

### Workflow Linting

```bash
# Install actionlint
brew install actionlint

# Lint workflows
actionlint .github/workflows/**/*.yml
```

## Versioning

This repository uses semantic versioning:

- `@v1` - Latest v1.x.x (recommended for consumers)
- `@v1.2` - Latest v1.2.x
- `@v1.2.3` - Specific version
- `@main` - Latest commit (development only)

## License

See [LICENSE](LICENSE) for details.

## Support

For issues or questions, please open an issue on GitHub.
