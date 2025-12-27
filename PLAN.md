# Reusable GitHub Actions Workflows for at-blacknight

## Executive Summary

This plan analyzes the SCA ci-templates reusable workflow architecture, identifies strengths and weaknesses, and proposes a comprehensive design for creating an at-blacknight reusable workflow repository. The first implementation will automate versioning for the A-Template-B Go application.

## 1. SCA ci-templates Analysis

### Strengths ✅

**Sophisticated Version Detection**
- [version.yml](../../../repos/ci-templates/.github/workflows/version.yml): Runs semantic-release in dry-run to forecast versions without side effects
- Rich output contract (8 outputs: version, would_release, release_type, etc.)
- Build-path filtering to skip builds when only docs change
- Supports multi-branch strategies (main, uat, develop)

**Separation of Concerns**
- version.yml (detection only) vs actual-release.yml (execution only)
- Safe for PR checks without creating releases

**Modern Docker Builds**
- [docker-build.yml](../../../repos/ci-templates/.github/workflows/docker-build.yml): Multi-platform (amd64/arm64), BuildKit, registry caching
- Smart tagging: stable releases → "latest", pre-releases → branch name

**Reusability Pattern**
- Consistent `workflow_call` trigger usage
- Well-defined input/output contracts

### Weaknesses ⚠️

**Documentation Void**
- README.md contains only "init" - no usage examples
- No explanation of how workflows compose together
- Difficult for new users to understand

**Complexity Overhead**
- version.yml is 435 lines with complex bash scripting
- Requires Node.js even for non-Node projects (Go, .NET)
- Steep learning curve (semantic-release + exec plugin approach)

**Architectural Issues**
- Two Docker workflows (docker-build.yml vs build-and-push-docker.yml) with unclear purpose
- Build-path filtering uses manual bash globs instead of GitHub's native path filtering
- No testing strategy for validating workflow changes
- Tight coupling to semantic-release v21.0.0

**Missing Features**
- No build-type specific workflows (Go, .NET, npm)
- No linting/security scanning patterns
- Limited deployment options (Helm only)
- No example consumer repositories

### Key Takeaways

**Keep:**
- workflow_call pattern for reusability
- Separation of version detection from release execution
- Multi-platform Docker builds
- Rich output contracts

**Improve:**
- Documentation (critical gap)
- Simplification for common cases
- Granular stages for composition
- Testing strategy

## 2. Proposed at-blacknight Architecture

### Design Philosophy

1. **Progressive complexity**: Simple workflows for simple cases, advanced features available when needed
2. **DRY through composition**: Reusable stages that orchestrate into complete workflows
3. **Build-type specific**: First-class support for Go, Docker, npm, .NET
4. **Well-documented**: Examples and usage docs for every workflow
5. **Testable**: Clear strategy for validating workflow changes

### Repository Structure

**User-Specified Structure:**
- `/workflows/` contains build-type specific workflows (go-release.yml, docker-release.yml, etc.)
- `/workflows/stages/` contains reusable components shared between build types
- All workflows use semantic versioning (version-detect dry-run + release-semantic actual)

```
at-blacknight/atb-ci/
├── .github/workflows/
│   │
│   ├── stages/                      # Shared reusable stages (DRY)
│   │   ├── version-detect.yml       # Semantic-release dry-run (forecast)
│   │   ├── release-semantic.yml     # Semantic-release actual (create tag/release)
│   │   ├── go-build-matrix.yml      # Go multi-platform build logic
│   │   ├── docker-build.yml         # Docker build + push logic
│   │   ├── npm-test.yml             # npm test + coverage logic
│   │   ├── npm-publish.yml          # npm publish logic
│   │   ├── dotnet-build.yml         # .NET build logic
│   │   └── dotnet-test.yml          # .NET test logic
│   │
│   ├── go-release.yml               # Go: version-detect → go-build → release-semantic
│   ├── docker-release.yml           # Docker: version-detect → docker-build → release-semantic
│   ├── npm-release.yml              # npm: version-detect → npm-test → npm-publish → release-semantic
│   └── dotnet-release.yml           # .NET: version-detect → dotnet-build → dotnet-test → release-semantic
│
├── docs/
│   ├── README.md                    # Overview and quick start
│   ├── architecture.md              # Design decisions and patterns
│   ├── conventional-commits.md      # Conventional commit guide
│   └── workflows/
│       ├── go-release.md            # Go workflow documentation
│       ├── docker-release.md        # Docker workflow documentation
│       ├── npm-release.md           # npm workflow documentation
│       └── stages.md                # Shared stages documentation
│
├── examples/
│   ├── go-cli-app/                  # Example: Go CLI app
│   │   └── .github/workflows/
│   │       └── ci.yml               # Consumer workflow example
│   ├── docker-app/                  # Example: Dockerized app
│   ├── npm-package/                 # Example: npm package
│   └── dotnet-app/                  # Example: .NET application
│
└── README.md                        # Main documentation
```

**Key Principles:**

1. **Shared Stages (DRY):**
   - `version-detect.yml` - Used by ALL build types for dry-run version detection
   - `release-semantic.yml` - Used by ALL build types for actual release creation
   - Build/test stages - Shared logic for each build type

2. **Build-Type Workflows:**
   - Each build type (Go, Docker, npm, .NET) has its own workflow file
   - All follow same pattern: version-detect → build/test → release-semantic
   - Customized for build-type specific requirements

3. **Always Semantic Versioning:**
   - No "simple" versioning option
   - All workflows use semantic-release
   - Conventional commits required

### Two-Tier Architecture

**Tier 1: Shared Stages** (in `/workflows/stages/`)
- Reusable components shared between build types
- Single responsibility, clear inputs/outputs
- **Common to all build types:**
  - `version-detect.yml` - Semantic-release dry-run (forecast version)
  - `release-semantic.yml` - Semantic-release actual (create tag/release)
- **Build-type specific:**
  - `go-build-matrix.yml` - Go builds
  - `docker-build.yml` - Docker builds
  - `npm-test.yml`, `npm-publish.yml` - npm stages
  - `dotnet-build.yml`, `dotnet-test.yml` - .NET stages

**Tier 2: Build-Type Workflows** (in `/workflows/`)
- One workflow per build type (go-release.yml, docker-release.yml, etc.)
- Orchestrate shared stages for specific build type
- All follow pattern: `version-detect → [build/test stages] → release-semantic`
- **THIS IS WHAT CONSUMER REPOS CALL** via `uses: at-blacknight/atb-ci/.github/workflows/go-release.yml@v1`
- Stages are internal implementation details, not exposed to consumers

**Example: go-release.yml structure**
```yaml
jobs:
  version:
    uses: ./.github/workflows/stages/version-detect.yml  # Shared

  build:
    needs: version
    if: needs.version.outputs.would-release == 'true'
    uses: ./.github/workflows/stages/go-build-matrix.yml  # Build-specific

  release:
    needs: [version, build]
    if: needs.version.outputs.would-release == 'true'
    uses: ./.github/workflows/stages/release-semantic.yml  # Shared
```

**Why This Structure:**
- `/workflows/stages/` = DRY components (version-detect, release-semantic shared across ALL build types)
- `/workflows/` = Build-type orchestration (go, docker, npm, .NET) - **THIS IS THE PUBLIC API**
- **Consumer repos call build-type workflows** (go-release.yml), **NOT stages directly**
- Stages are implementation details, hidden from consumers
- Maximal code reuse between build types

**Call Hierarchy:**
```
Consumer Repo (e.g., A-Template-B)
  └─ calls: atb-ci/.github/workflows/go-release.yml@v1
       ├─ internally calls: stages/version-detect.yml
       ├─ internally calls: stages/go-build-matrix.yml
       └─ internally calls: stages/release-semantic.yml
```

**Why This Matters:**
- Consumer repos stay simple (single workflow reference)
- Can refactor stages without breaking consumers
- Versioning is cleaner (@v1 applies to whole pipeline)
- Encapsulation: stages are implementation details

### Versioning Strategy

**Semantic Versioning Only**

**User Decision:** Always use semantic versioning, no simple/manual versioning option

**Implementation:**
- All workflows use semantic-release
- Two-stage approach (from SCA pattern):
  1. **Dry-run stage** (`version-detect.yml`) - Forecast version, safe for PRs
  2. **Actual release stage** (`release-semantic.yml`) - Create tags/releases
- Uses conventional commits exclusively
- Automatic version bumping based on commit types
- Automatic changelog generation

**Conventional Commit Format:**
```
<type>[optional scope]: <description>

[optional body]

[optional footer]
```

**Types:**
- `feat:` - New feature (minor version bump)
- `fix:` - Bug fix (patch version bump)
- `docs:` - Documentation only (no release)
- `style:` - Code style changes (no release)
- `refactor:` - Code refactoring (no release)
- `perf:` - Performance improvement (patch version bump)
- `test:` - Adding tests (no release)
- `chore:` - Maintenance tasks (no release)
- `feat!:` or `fix!:` - Breaking change (major version bump)

## 3. First Implementation: Go Versioning for A-Template-B

### Current State

**A-Template-B build.yaml** (179 lines):
- Manual tag push triggers release
- 6-platform matrix build (linux/windows/darwin × amd64/arm64)
- Windows resource embedding with goversioninfo
- Version injection via ldflags: `-X main.version={{tag}}`
- GitHub release with auto-generated notes
- Pre-release detection (v1.2.3-rc1)

### Implementation Approach: Semantic Versioning First

**Goal:** Jump straight to conventional commits and automatic version bumping

**Decision Rationale:**
- User selected semantic versioning immediately
- Requires discipline with commit messages
- Fully automated releases from day one
- No need for manual tag creation

**New Consumer Workflow** (.github/workflows/ci.yml):
```yaml
name: CI
on:
  push:
    branches: [main]
  pull_request:

jobs:
  # Single job: calls go-release.yml which handles everything
  release:
    uses: at-blacknight/atb-ci/.github/workflows/go-release.yml@v1
    with:
      go-version: '1.24'
      binary-name: 'ATemplateB'
      enable-windows-resource: true
      windows-company-name: 'AT-BlacKnight'
      windows-product-name: 'ATemplateB'
    permissions:
      contents: write
```

**Note:** Consumer repos call **go-release.yml** (the build-type workflow), NOT the individual stages. The go-release.yml workflow internally handles:
1. Calling version-detect.yml (dry-run on PRs, actual on main)
2. Calling go-build-matrix.yml (if would-release=true)
3. Calling release-semantic.yml (if would-release=true)

This keeps consumer workflows simple and encapsulates the orchestration logic.

**Result:** Fully automated releases based on conventional commits

**Conventional Commit Examples:**
```bash
git commit -m "feat: add new template function"      # → 1.2.0 → 1.3.0
git commit -m "fix: resolve parsing error"           # → 1.2.0 → 1.2.1
git commit -m "feat!: breaking API change"           # → 1.2.0 → 2.0.0
git commit -m "docs: update README"                  # → No release
```

## 4. Critical Workflows to Build

### Shared Stages (Used by ALL Build Types)

These stages live in `.github/workflows/stages/` and are shared across all build-type workflows.

### 1. Stage: version-detect.yml (Shared - Dry-Run)

**Purpose:** Semantic-release dry-run to forecast version without side effects

**Path:** `.github/workflows/stages/version-detect.yml`

**Used By:** ALL build types (go-release.yml, docker-release.yml, npm-release.yml, dotnet-release.yml)

**Inputs:**
- `node-version`: Node.js version for semantic-release (default: "20.x")
- `upload-artifacts`: Whether to upload changelog artifacts (default: true)

**Outputs:**
- `version`: Next semantic version (e.g., "1.2.3")
- `would-release`: Whether a release would be created (boolean)
- `release-type`: Type of release (major, minor, patch)
- `release-channel`: Release channel (main, uat, develop)
- `git-tag`: Git tag name that will be created (e.g., "v1.2.3")
- `release-notes`: Generated changelog content

**Key Features:**
- Runs semantic-release in dry-run mode (safe for PRs)
- Uses conventional commits to determine version bump
- Supports multi-branch strategies (main, uat, develop)
- Generates changelog automatically

### 2. Stage: release-semantic.yml (Shared - Actual Release)

**Purpose:** Semantic-release actual execution to create tags, releases, and commit changelog

**Path:** `.github/workflows/stages/release-semantic.yml`

**Used By:** ALL build types (go-release.yml, docker-release.yml, npm-release.yml, dotnet-release.yml)

**Inputs:**
- `node-version`: Node.js version (default: "20.x")
- `git-user-name`: Git committer name (default: "Release Bot")
- `git-user-email`: Git committer email (default: "release-bot@github.com")

**Actions:**
- Runs semantic-release for real (not dry-run)
- Creates git tags
- Creates GitHub releases
- Commits CHANGELOG.md back to repository
- Attaches build artifacts to release

**Key Features:**
- Uses semantic-release github plugin
- Uses semantic-release git plugin to commit changelog
- Handles pre-releases (uat, develop branches)
- Attaches downloaded artifacts from build job

### Build-Type Specific Stages

These stages live in `.github/workflows/stages/` but are specific to each build type.

### 3. Stage: go-build-matrix.yml (Go-Specific)

**Purpose:** Multi-platform Go builds

**Path:** `.github/workflows/stages/go-build-matrix.yml`

**Used By:** go-release.yml only

**Inputs:**
- `go-version`: Go version (default: "1.24")
- `version`: Version to inject via ldflags
- `platforms`: Comma-separated list (default: 6 platforms)
- `binary-name`: Binary name (required)
- `enable-windows-resource`: Enable goversioninfo (default: false)
- `windows-company-name`: Company name for Windows metadata
- `windows-product-name`: Product name for Windows metadata

**Outputs:**
- Uploads build artifacts for each platform

### Build-Type Orchestration Workflows

These workflows live in `.github/workflows/` and orchestrate stages for each build type.

### 4. Workflow: go-release.yml (Go Build Type) - PUBLIC API

**Purpose:** Full Go release pipeline with semantic versioning - **THIS IS WHAT CONSUMER REPOS CALL**

**Path:** `.github/workflows/go-release.yml`

**Called By:** Consumer repositories (e.g., A-Template-B, example-go-app)

**Inputs:**
- `go-version`: Go version (default: "1.24")
- `platforms`: Platform list (default: "linux/amd64,linux/arm64,darwin/amd64,darwin/arm64,windows/amd64,windows/arm64")
- `binary-name`: Binary name (required)
- `enable-windows-resource`: Windows metadata (default: false)
- `windows-company-name`: Company for Windows .exe metadata
- `windows-product-name`: Product for Windows .exe metadata

**Internal Implementation (calls stages):**
```yaml
# This is the INTERNAL logic of go-release.yml
# Consumer repos don't need to know about this

jobs:
  version:
    uses: ./.github/workflows/stages/version-detect.yml  # Internal call

  build:
    needs: version
    if: needs.version.outputs.would-release == 'true'
    uses: ./.github/workflows/stages/go-build-matrix.yml  # Internal call
    with:
      version: ${{ needs.version.outputs.version }}

  release:
    needs: [version, build]
    if: needs.version.outputs.would-release == 'true'
    uses: ./.github/workflows/stages/release-semantic.yml  # Internal call
```

**Consumer Usage:**
```yaml
# A-Template-B/.github/workflows/ci.yml
jobs:
  release:
    uses: at-blacknight/atb-ci/.github/workflows/go-release.yml@v1  # Call this!
    with:
      binary-name: 'ATemplateB'
      # ... other inputs
```

**Design Note:** Consumer repos call `go-release.yml` (the public API), not the internal stages. This provides encapsulation and allows refactoring stages without breaking consumers.

**Key Features:**
- Skips build/release if no releasable changes (would-release=false)
- Passes version through job outputs
- Conditional execution based on semantic-release forecast
- Calls shared stages (version-detect, release-semantic)
- Calls Go-specific stage (go-build-matrix)

**Pattern for All Build Types:**

All build-type workflows follow this pattern:

```yaml
# Pattern: <buildtype>-release.yml
jobs:
  version:
    uses: ./.github/workflows/stages/version-detect.yml       # SHARED

  build:
    needs: version
    if: needs.version.outputs.would-release == 'true'
    uses: ./.github/workflows/stages/<buildtype>-build.yml    # BUILD-SPECIFIC

  release:
    needs: [version, build]
    if: needs.version.outputs.would-release == 'true'
    uses: ./.github/workflows/stages/release-semantic.yml     # SHARED
```

This ensures:
- DRY: version-detect and release-semantic reused across ALL build types
- Consistency: All workflows follow same pattern
- Flexibility: Build-specific logic isolated to build stages

## 5. Future Expansion

### Docker Builds

**Stages:**
- `stages/docker-build.yml` - Multi-platform Docker build + push logic

**Workflow:**
- `docker-release.yml` - Orchestrates: version-detect → docker-build → release-semantic

**Features:**
- Multi-platform builds (amd64, arm64)
- Smart tagging (latest, semver, branch)
- Registry caching
- Support for GHCR, Docker Hub

**Usage:**
```yaml
jobs:
  docker:
    uses: at-blacknight/atb-ci/.github/workflows/docker-release.yml@v1
    with:
      registry: ghcr.io
      image-name: a-template-b
      platforms: 'linux/amd64,linux/arm64'
```

**Follows Same Pattern:**
```yaml
# docker-release.yml
jobs:
  version:
    uses: ./.github/workflows/stages/version-detect.yml       # SHARED

  build:
    uses: ./.github/workflows/stages/docker-build.yml         # DOCKER-SPECIFIC

  release:
    uses: ./.github/workflows/stages/release-semantic.yml     # SHARED
```

### npm Packages

**Stages:**
- `stages/npm-test.yml` - npm test + coverage logic
- `stages/npm-publish.yml` - npm publish logic

**Workflow:**
- `npm-release.yml` - Orchestrates: version-detect → npm-test → npm-publish → release-semantic

**Features:**
- npm test + coverage
- Publish to npm or GitHub Packages
- Support for scoped packages
- Pre-release tags

**Follows Same Pattern:**
```yaml
# npm-release.yml
jobs:
  version:
    uses: ./.github/workflows/stages/version-detect.yml       # SHARED

  test:
    uses: ./.github/workflows/stages/npm-test.yml             # NPM-SPECIFIC

  publish:
    uses: ./.github/workflows/stages/npm-publish.yml          # NPM-SPECIFIC

  release:
    uses: ./.github/workflows/stages/release-semantic.yml     # SHARED
```

### .NET Applications

**Stages:**
- `stages/dotnet-build.yml` - Multi-framework build + NuGet package logic
- `stages/dotnet-test.yml` - dotnet test + coverage logic

**Workflow:**
- `dotnet-release.yml` - Orchestrates: version-detect → dotnet-build → dotnet-test → release-semantic

**Features:**
- Multi-framework builds (net8.0, net6.0)
- NuGet package creation
- Test + coverage
- Platform-specific builds

**Follows Same Pattern:**
```yaml
# dotnet-release.yml
jobs:
  version:
    uses: ./.github/workflows/stages/version-detect.yml       # SHARED

  build:
    uses: ./.github/workflows/stages/dotnet-build.yml         # DOTNET-SPECIFIC

  test:
    uses: ./.github/workflows/stages/dotnet-test.yml          # DOTNET-SPECIFIC

  release:
    uses: ./.github/workflows/stages/release-semantic.yml     # SHARED
```

### Deployment Options

**deploy-helm.yml:**
- Kubernetes/Helm deployments
- Multi-environment support (dev, staging, prod)
- Dry-run validation

**deploy-cloud-run.yml:**
- Google Cloud Run deployments
- Multi-region support

**deploy-ssh.yml:**
- SSH-based deployments to VMs
- Binary distribution
- Health checks

## 6. Testing Strategy

### Multi-Level Testing

**Level 1: Lint and Validate**
- actionlint for workflow syntax
- yamllint for YAML formatting
- Runs on every PR

**Level 2: Integration Tests**
- Test workflows run on changes to workflow files
- Use minimal sample apps in tests/ directory

**Level 3: Example Repositories**
- Create example repos (at-blacknight/example-go-app)
- Use atb-ci@main during development
- Validates end-to-end functionality

**Level 4: Versioned Releases**
- atb-ci uses semantic versioning
- Consumers pin to @v1, @v1.2, or @v1.2.3
- Breaking changes require major version bump

## 7. Implementation Timeline

**User Preferences Applied:**
- ✅ Start with semantic versioning (skip simple versioning)
- ✅ Create example apps first (test before A-Template-B migration)
- ✅ Use stages/ subdirectory for atomic workflows
- ✅ Prioritize: Go first, then Docker, npm, .NET

### Phase 1: Foundation with Semantic Versioning (Week 1)

**Deliverables:**
1. Create at-blacknight/atb-ci repository
2. Implement **shared stages** (used by all build types):
   - `.github/workflows/stages/version-detect.yml` (semantic-release dry-run)
   - `.github/workflows/stages/release-semantic.yml` (semantic-release actual)
3. Implement **Go-specific stages**:
   - `.github/workflows/stages/go-build-matrix.yml` (multi-platform Go builds)
4. Implement **Go workflow** (orchestration):
   - `.github/workflows/go-release.yml` (version-detect → go-build-matrix → release-semantic)
5. Create example-go-app repository for testing
6. Basic documentation (README, architecture.md, conventional-commits.md)

**Success Criteria:**
- Example Go app builds and releases automatically on main push
- Versions bump correctly based on conventional commits
- All 6 platforms build successfully

### Phase 2: Documentation and Testing (Week 2)

**Deliverables:**
1. Comprehensive documentation:
   - docs/workflows/go-release.md
   - docs/workflows/version-detection.md
   - Conventional commit guide
2. Workflow linting setup (actionlint)
3. Integration tests for Go workflows
4. A-Template-B migration guide (ready but not executed)

**Success Criteria:**
- All workflows pass linting
- Documentation covers all common use cases
- Example app runs successfully

### Phase 3: Docker Support (Week 3)

**Deliverables:**
1. Implement **Docker-specific stage**:
   - `.github/workflows/stages/docker-build.yml` (multi-platform build + push)
2. Implement **Docker workflow** (orchestration):
   - `.github/workflows/docker-release.yml` (version-detect → docker-build → release-semantic)
3. Multi-platform support (amd64, arm64)
4. Registry support (GHCR, Docker Hub)
5. Create example-docker-app repository
6. Docker workflow documentation

**Success Criteria:**
- Docker images build for multiple platforms
- Smart tagging works (latest, semver, branch)
- Uses shared version-detect and release-semantic stages
- Example Docker app deploys successfully

### Phase 4: npm Support (Week 4)

**Deliverables:**
1. Implement **npm-specific stages**:
   - `.github/workflows/stages/npm-test.yml` (test + coverage)
   - `.github/workflows/stages/npm-publish.yml` (publish to npm/GitHub)
2. Implement **npm workflow** (orchestration):
   - `.github/workflows/npm-release.yml` (version-detect → npm-test → npm-publish → release-semantic)
3. Create example-npm-package repository
4. npm workflow documentation

**Success Criteria:**
- npm packages publish correctly
- Tests run and coverage reported
- Uses shared version-detect and release-semantic stages

### Phase 5: .NET Support (Week 5)

**Deliverables:**
1. Implement **.NET-specific stages**:
   - `.github/workflows/stages/dotnet-build.yml` (multi-framework build + NuGet)
   - `.github/workflows/stages/dotnet-test.yml` (test + coverage)
2. Implement **.NET workflow** (orchestration):
   - `.github/workflows/dotnet-release.yml` (version-detect → dotnet-build → dotnet-test → release-semantic)
3. Create example-dotnet-app repository
4. .NET workflow documentation

**Success Criteria:**
- .NET packages build for multiple frameworks
- Tests run and coverage reported
- Uses shared version-detect and release-semantic stages

### Phase 6: Polish and v1.0.0 Release (Week 6)

**Deliverables:**
1. Review and refine all workflows
2. Comprehensive integration testing
3. Final documentation review
4. Release atb-ci@v1.0.0
5. **Now ready to migrate A-Template-B**

## 8. Migration Plan for A-Template-B

**Note:** Migration happens AFTER atb-ci@v1.0.0 is released and tested with example apps (Phase 6).

### Prerequisites

1. ✅ atb-ci@v1.0.0 released and tested
2. ✅ Example Go app successfully using semantic versioning
3. ✅ Documentation complete
4. ✅ Team trained on conventional commits

### Migration Steps

### Step 1: Prepare Repository

**1a. Review recent commits**
```bash
cd A-Template-B
git log --oneline -20
```

Check if commits already follow conventional format. If not, proceed with caution.

**1b. Create backup branch**
```bash
git checkout -b backup/manual-workflow
git push origin backup/manual-workflow
git checkout main
```

### Step 2: Update Workflow

**2a. Remove old workflow**
```bash
mv .github/workflows/build.yaml .github/workflows/build.yaml.backup
```

**2b. Create new CI workflow**

Create `.github/workflows/ci.yml`:
```yaml
name: CI
on:
  push:
    branches: [main]
  pull_request:

permissions:
  contents: write

jobs:
  # Single job: calls go-release.yml
  # go-release.yml handles version-detect, build, and release internally
  release:
    uses: at-blacknight/atb-ci/.github/workflows/go-release.yml@v1
    with:
      go-version: '1.24'
      binary-name: 'ATemplateB'
      platforms: 'linux/amd64,linux/arm64,darwin/amd64,darwin/arm64,windows/amd64,windows/arm64'
      enable-windows-resource: true
      windows-company-name: 'AT-BlacKnight'
      windows-product-name: 'ATemplateB'
```

**Key Design Decision:**
- Consumer repos call the **build-type workflow** (go-release.yml)
- They do **NOT** call individual stages (version-detect.yml, go-build-matrix.yml, release-semantic.yml)
- Stages are **internal implementation details** of the build-type workflow
- This provides **encapsulation** - we can refactor stages without breaking consumers

### Step 3: Test with PR First

**3a. Create test branch**
```bash
git checkout -b feat/test-semantic-versioning
git add .github/workflows/ci.yml
git commit -m "feat: migrate to semantic versioning workflow"
git push origin feat/test-semantic-versioning
```

**3b. Create PR and verify**
- PR check should run version-check job
- Should show "would release: true" and next version (e.g., 1.3.0)
- Verify it DOES NOT create a release (dry-run only)

### Step 4: Merge and Release

**4a. Merge PR**
```bash
# After PR review and approval
git checkout main
git merge feat/test-semantic-versioning
git push origin main
```

**4b. Monitor release workflow**
- Watch GitHub Actions run on main push
- Verify builds complete for all 6 platforms
- Verify release is created with correct version
- Verify changelog is generated and committed

**4c. Verify release artifacts**
- Download release assets
- Check ZIP files exist for all platforms
- Verify checksums
- Test Windows .exe has version metadata

### Step 5: Update Documentation

**5a. Update README.md**

Add section:
```markdown
## Releasing

This project uses semantic versioning with conventional commits.

### Commit Message Format
- `feat: add feature` → Minor bump (1.2.0 → 1.3.0)
- `fix: bug fix` → Patch bump (1.2.0 → 1.2.1)
- `feat!: breaking change` → Major bump (1.2.0 → 2.0.0)
- `docs: documentation` → No release

### Release Process
Releases are automatic when merging to main. See CHANGELOG.md for history.
```

**5b. Remove old build instructions**

Remove any manual tagging instructions from docs.

### Step 6: Cleanup

**6a. Remove backup workflow**
```bash
rm .github/workflows/build.yaml.backup
git add .github/workflows/
git commit -m "chore: remove old build workflow"
git push
```

**6b. Delete backup branch (after confirming stability)**
```bash
# Wait 1-2 weeks, ensure releases work correctly
git branch -D backup/manual-workflow
git push origin --delete backup/manual-workflow
```

### Rollback Plan

**If semantic versioning doesn't work:**

**Option 1: Quick rollback (restore old workflow)**
```bash
git checkout backup/manual-workflow -- .github/workflows/build.yaml
rm .github/workflows/ci.yml
git commit -m "revert: restore manual tagging workflow"
git push
```

**Option 2: Keep reusable workflow but switch to simple versioning**

Add stages/version-simple.yml to atb-ci and update A-Template-B to use it:
```yaml
# Use simple versioning strategy
jobs:
  release:
    uses: at-blacknight/atb-ci/.github/workflows/go-release.yml@v1
    with:
      versioning-strategy: 'simple'  # Falls back to manual tags
      ...
```

### Post-Migration Checklist

- ✅ First automated release created successfully
- ✅ All platforms built (6 binaries)
- ✅ Windows metadata embedded correctly
- ✅ CHANGELOG.md generated and committed
- ✅ Documentation updated
- ✅ Team understands conventional commits
- ✅ Backup workflow can be safely deleted

## 9. Key Design Decisions

### Consumer API: Call Build-Type Workflows, Not Stages

**Decision:** Consumer repos call build-type workflows (go-release.yml), NOT individual stages

**Rationale:**
- **Simplicity:** Consumer workflows are 10-20 lines, not 50+
- **Encapsulation:** Stages are implementation details, can be refactored without breaking consumers
- **Versioning:** @v1 applies to entire pipeline, not individual stages
- **Maintainability:** Stage composition changes don't affect consumers
- **Consistency:** All consumer repos follow same pattern

**Example:**
```yaml
# GOOD: Consumer calls build-type workflow
jobs:
  release:
    uses: at-blacknight/atb-ci/.github/workflows/go-release.yml@v1

# BAD: Consumer calls stages directly
jobs:
  version:
    uses: at-blacknight/atb-ci/.github/workflows/stages/version-detect.yml@v1
  build:
    uses: at-blacknight/atb-ci/.github/workflows/stages/go-build-matrix.yml@v1
  release:
    uses: at-blacknight/atb-ci/.github/workflows/stages/release-semantic.yml@v1
```

**Exception:** Advanced users can compose stages directly if they need custom orchestration, but this is not the recommended pattern.

### Monolithic vs Granular Stages

**Decision:** Two-tier with shared stages

**Rationale:**
- Shared stages (version-detect, release-semantic) reused across ALL build types (DRY)
- Build-specific stages (go-build-matrix, docker-build) isolated
- Build-type workflows orchestrate stages (encapsulation)

### Branch Strategy

**Decision:** Support multiple strategies

**Default:** Trunk-based (main only) - simplest

**Also supports:**
- Git Flow (main + develop + release branches)
- Environment-based (main + uat + develop)

### Semantic-Release Configuration

**Decision:** Embed config in workflow for v1

**Future:** Extract to shareable configs for v2

**Rationale:** Embedding is simpler and reduces moving parts for initial adoption

## 10. Success Criteria

**For A-Template-B Migration:**
- ✅ Release creation automated
- ✅ All platforms build successfully
- ✅ Windows metadata preserved
- ✅ Workflow reduced from 179 to ~16 lines
- ✅ Easy rollback if issues found

**For atb-ci Repository:**
- ✅ Well-documented with examples
- ✅ All workflows pass linting
- ✅ Integration tests passing
- ✅ Example repos build successfully (Go, Docker, npm, .NET)
- ✅ All build types use shared version-detect and release-semantic stages
- ✅ Ready for v1.0.0 release

**Architecture Validation:**
- ✅ Shared stages (version-detect, release-semantic) work across all build types
- ✅ Build-specific stages (go-build-matrix, docker-build, etc.) properly isolated
- ✅ All workflows follow consistent pattern
- ✅ No code duplication between build types
- ✅ DRY principle achieved

## Critical Files Reference

**SCA ci-templates (for reference):**
- [version.yml](../../../repos/ci-templates/.github/workflows/version.yml) - Version detection pattern
- [actual-release.yml](../../../repos/ci-templates/.github/workflows/actual-release.yml) - Release execution pattern
- [docker-build.yml](../../../repos/ci-templates/.github/workflows/docker-build.yml) - Multi-platform Docker builds

**at-blacknight A-Template-B (to migrate):**
- [build.yaml](../../../repos/A-Template-B/.github/workflows/build.yaml) - Current build workflow
- [ATemplateB.go](../../../repos/A-Template-B/ATemplateB.go#L34) - Version variable injection

