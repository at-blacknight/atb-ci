# Architecture Overview

This document explains the design decisions and architecture of the atb-ci reusable workflows.

## Design Philosophy

1. **Progressive Complexity**: Simple workflows for simple cases, advanced features available when needed
2. **DRY Through Composition**: Reusable stages orchestrated into complete workflows
3. **Build-Type Specific**: First-class support for Go, Docker, npm, .NET
4. **Well-Documented**: Examples and usage docs for every workflow
5. **Semantic Versioning Only**: All workflows use conventional commits and semantic-release

## Two-Tier Architecture

### Tier 1: Shared Stages (`.github/workflows/stage-*.yml`)

Atomic, reusable components shared between build types (prefixed with `stage-`).

**Note**: Due to GitHub Actions limitations, reusable workflows must be in the root `.github/workflows/` directory, not in subdirectories. We use the `stage-` prefix to indicate these are internal building blocks.

#### Shared Across All Build Types
- **`stage-version-detect.yml`** - Semantic-release dry-run (forecast version)
- **`stage-release-semantic.yml`** - Semantic-release actual (create tag/release)

#### Build-Type Specific
- **`stage-go-build-matrix.yml`** - Multi-platform Go builds
- **`stage-docker-build.yml`** - Multi-platform Docker builds (coming soon)
- **`stage-npm-test.yml`** - npm test + coverage (coming soon)
- **`stage-npm-publish.yml`** - npm publish (coming soon)
- **`stage-dotnet-build.yml`** - .NET builds (coming soon)
- **`stage-dotnet-test.yml`** - .NET tests (coming soon)

### Tier 2: Build-Type Workflows (`.github/workflows/`)

Orchestrate shared stages for specific build types. **This is the public API** that consumer repositories call.

- **`go-release.yml`** - Go: version-detect → go-build → release-semantic
- **`docker-release.yml`** - Docker: version-detect → docker-build → release-semantic (coming soon)
- **`npm-release.yml`** - npm: version-detect → npm-test → npm-publish → release-semantic (coming soon)
- **`dotnet-release.yml`** - .NET: version-detect → dotnet-build → dotnet-test → release-semantic (coming soon)

## Consumer API

**Consumer repos call build-type workflows**, NOT individual stages.

### Good Example ✅
```yaml
# Consumer repo: A-Template-B/.github/workflows/ci.yml
jobs:
  release:
    uses: at-blacknight/atb-ci/.github/workflows/go-release.yml@v1
    with:
      binary-name: 'ATemplateB'
```

### Bad Example ❌
```yaml
# DON'T do this - stages are internal implementation details
jobs:
  version:
    uses: at-blacknight/atb-ci/.github/workflows/stages/version-detect.yml@v1
  build:
    uses: at-blacknight/atb-ci/.github/workflows/stages/go-build-matrix.yml@v1
  release:
    uses: at-blacknight/atb-ci/.github/workflows/stages/release-semantic.yml@v1
```

### Why?

1. **Simplicity**: Consumer workflows are 10-20 lines, not 50+
2. **Encapsulation**: Stages are implementation details, can be refactored without breaking consumers
3. **Versioning**: `@v1` applies to entire pipeline, not individual stages
4. **Maintainability**: Stage composition changes don't affect consumers
5. **Consistency**: All consumer repos follow same pattern

## Call Hierarchy

```
Consumer Repo (e.g., A-Template-B)
  └─ calls: atb-ci/.github/workflows/go-release.yml@v1
       ├─ internally calls: stage-version-detect.yml
       ├─ internally calls: stage-go-build-matrix.yml
       └─ internally calls: stage-release-semantic.yml
```

## Workflow Pattern

All build-type workflows follow this pattern:

```yaml
jobs:
  version:
    uses: ./.github/workflows/stage-version-detect.yml       # SHARED

  build:
    needs: version
    if: needs.version.outputs.would-release == 'true'
    uses: ./.github/workflows/stage-<buildtype>-build.yml    # BUILD-SPECIFIC

  release:
    needs: [version, build]
    if: needs.version.outputs.would-release == 'true'
    uses: ./.github/workflows/stage-release-semantic.yml     # SHARED
```

This ensures:
- **DRY**: version-detect and release-semantic reused across ALL build types
- **Consistency**: All workflows follow same pattern
- **Flexibility**: Build-specific logic isolated to build stages

## Semantic Versioning Flow

### On Pull Requests (Dry-Run)

1. **version-detect.yml** runs semantic-release in dry-run mode
2. Forecasts next version based on commits
3. Shows version in PR checks
4. **Does NOT create release** (safe for PRs)

**Output:**
```
Would release: true
Next version: 1.3.0
Release type: minor
```

### On Main Branch Push (Actual Release)

1. **version-detect.yml** - Forecast version (same as PR)
2. **build stage** - Build binaries/images (conditional on would-release=true)
3. **release-semantic.yml** - Create actual release:
   - Creates git tag (e.g., v1.3.0)
   - Creates GitHub release with changelog
   - Attaches build artifacts
   - Commits CHANGELOG.md back to repo

## Conventional Commits

Version bumps are determined by commit types:

- `feat:` → Minor bump (1.2.0 → 1.3.0)
- `fix:` → Patch bump (1.2.0 → 1.2.1)
- `feat!:` → Major bump (1.2.0 → 2.0.0)
- `docs:`, `chore:`, etc. → No release

See [Conventional Commits Guide](conventional-commits.md) for details.

## Branch Strategies

### Default: Trunk-Based (Main Only)

Simplest strategy for most projects.

**Branches:**
- `main` - Stable releases (1.0.0, 1.1.0, 1.2.0)

**Workflow:**
```
main: feat → 1.0.0 → 1.1.0
```

### Git Flow

Traditional Git Flow with release branches.

**Branches:**
- `main` - Stable releases (1.0.0, 1.1.0)
- `develop` - Beta releases (1.1.0-beta.1)
- `release/*` - Release candidates (1.1.0-rc.1)

**Workflow:**
```
develop: feat → 1.1.0-beta.1
release/1.1: fix → 1.1.0-rc.1
main: merge → 1.1.0
```

### Environment-Based

Environment-specific pre-releases.

**Branches:**
- `main` - Production (1.0.0)
- `uat` - UAT pre-releases (1.1.0-rc.1)
- `develop` - Dev pre-releases (1.1.0-beta.1)

**Workflow:**
```
develop: feat → 1.1.0-beta.1 → deploy to dev
uat: merge → 1.1.0-rc.1 → deploy to UAT
main: merge → 1.1.0 → deploy to prod
```

## Go Build Workflow Details

### Multi-Platform Builds

The `go-build-matrix.yml` stage builds for multiple platforms:

**Default Platforms:**
- linux/amd64
- linux/arm64
- darwin/amd64 (macOS Intel)
- darwin/arm64 (macOS Apple Silicon)
- windows/amd64
- windows/arm64

**Customizable:**
```yaml
with:
  platforms: 'linux/amd64,windows/amd64'  # Only these two
```

### Version Injection

Version is injected via ldflags:

**Default:**
```go
// main.go
var version = "dev"  // Replaced at build time

// Command:
go build -ldflags "-X main.version=1.2.3"
```

**Customizable:**
```yaml
with:
  ldflags-template: '-X github.com/user/app/cmd.Version={{version}}'
```

### Windows Resource Embedding

When `enable-windows-resource: true`, the workflow:

1. Installs goversioninfo
2. Generates versioninfo.json with metadata
3. Embeds resources in .exe

**Result:** Windows executables have proper version metadata visible in Properties.

### Build Artifacts

Each platform produces:
- `{binary}-{version}-{os}-{arch}.zip` - Compressed binary
- `{binary}-{version}-{os}-{arch}.zip.sha256` - Checksum

**Example:**
```
ATemplateB-1.2.3-linux-amd64.zip
ATemplateB-1.2.3-linux-amd64.zip.sha256
ATemplateB-1.2.3-windows-amd64.zip
ATemplateB-1.2.3-windows-amd64.zip.sha256
```

## Semantic Release Configuration

### Plugins Used

1. **@semantic-release/commit-analyzer** - Analyze commits for version bumps
2. **@semantic-release/release-notes-generator** - Generate changelog
3. **@semantic-release/changelog** - Update CHANGELOG.md
4. **@semantic-release/github** - Create GitHub release + attach artifacts
5. **@semantic-release/git** - Commit CHANGELOG.md back to repo

### Configuration Location

**v1.x**: Embedded in workflows (`.releaserc.json` generated at runtime)

**Future (v2.x)**: Shareable configs for advanced customization

## Testing Strategy

### Level 1: Lint and Validate
- actionlint for workflow syntax
- yamllint for YAML formatting
- Runs on every PR

### Level 2: Integration Tests
- Test workflows run on changes to workflow files
- Use minimal sample apps in tests/ directory

### Level 3: Example Repositories
- Create example repos (at-blacknight/example-go-app)
- Use atb-ci@main during development
- Validates end-to-end functionality

### Level 4: Versioned Releases
- atb-ci uses semantic versioning
- Consumers pin to @v1, @v1.2, or @v1.2.3
- Breaking changes require major version bump

## Security Considerations

### Permissions

Workflows require minimal permissions:

**Required:**
- `contents: write` - For creating releases and committing CHANGELOG.md

**Not Required:**
- No admin permissions
- No secrets beyond GITHUB_TOKEN

### Token Usage

All workflows use the automatic `github.token`:
- No PAT (Personal Access Token) required
- Scoped to repository
- Expires after workflow run

### Supply Chain Security

- Pin action versions (e.g., `actions/checkout@v4`)
- Use semantic-release official plugins only
- No third-party build scripts
- Reproducible builds (same commits = same artifacts)

## Future Enhancements

### Phase 2: Docker Support (Week 3)
- Multi-platform Docker builds
- Smart tagging (latest, semver, branch)
- Registry support (GHCR, Docker Hub)

### Phase 3: npm Support (Week 4)
- npm test + coverage
- Publish to npm or GitHub Packages
- Support for scoped packages

### Phase 4: .NET Support (Week 5)
- Multi-framework builds
- NuGet package creation
- Test + coverage

### Phase 5: Deployment Workflows
- Kubernetes/Helm deployments
- Cloud Run deployments
- SSH-based deployments

## Contributing

### Adding New Build Types

1. Create build-specific stages in `stages/`
2. Create orchestration workflow in root
3. Follow existing pattern (version → build → release)
4. Add documentation
5. Create example repository
6. Add integration tests

### Modifying Existing Workflows

1. Update workflow file
2. Update documentation
3. Test with example repositories
4. Consider backward compatibility
5. Update CHANGELOG.md

## Resources

- [Semantic Release Documentation](https://semantic-release.gitbook.io/)
- [Conventional Commits](https://www.conventionalcommits.org/)
- [GitHub Actions Reusable Workflows](https://docs.github.com/en/actions/using-workflows/reusing-workflows)
- [Semantic Versioning](https://semver.org/)
