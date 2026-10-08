---
type: "Reference"
title: "CI/CD Workflows: GitHub Actions and Release Process"
description: "LangChain's GitHub Actions-based CI/CD system automating testing, linting, and release management across a monorepo with intelligent change detection, parallel matrix testing, and strict release gates."
tags: [ci-cd, github-actions, testing, linting, release, pypi, monorepo, automation]
sources:
  - id: openwiki-source-3638ff2c170b760df159d81b
    resource: repo://.github/actions/uv_setup/action.yml
  - id: openwiki-source-34e57b5a3a0c875639ab72a7
    resource: repo://.github/scripts/check_diff.py
  - id: openwiki-source-f35e7c44cc1805709393a581
    resource: repo://.github/workflows/_lint.yml
  - id: openwiki-source-c92cc62695c6def991956428
    resource: repo://.github/workflows/_release.yml
  - id: openwiki-source-c9c292f4ecabe180cdae27ce
    resource: repo://.github/workflows/_test_pydantic.yml
  - id: openwiki-source-d8a8900818f4abab719bd1b7
    resource: repo://.github/workflows/_test_vcr.yml
  - id: openwiki-source-4d9cccca7700db7220ec055e
    resource: repo://.github/workflows/_test.yml
  - id: openwiki-source-7330cb37457ccdb62d7c41c7
    resource: repo://.github/workflows/auto-label-by-package.yml
  - id: openwiki-source-6e3a52c89729b5704dbd7eec
    resource: repo://.github/workflows/check_diffs.yml
  - id: openwiki-source-9069a5dd5fbb579fbd5470ce
    resource: repo://.github/workflows/integration_tests.yml
  - id: openwiki-source-6d4b4e707b8d60b6ccfa3425
    resource: repo://.github/workflows/openwiki-update.yml
  - id: openwiki-source-f8781d847f6481a966a44a68
    resource: repo://.github/workflows/pr_labeler.yml
  - id: openwiki-source-12805fbf767dc2a3e238645e
    resource: repo://.github/workflows/pr_lint.yml
  - id: openwiki-source-4d1645cb6317345817452838
    resource: repo://.pre-commit-config.yaml
generated: { by: "openwiki/0.5.0", at: "2026-10-08T08:29:58.787Z" }
verified:
  - by: openwiki/0.5.0
    at: 2026-10-08T08:29:58.787Z
---

# CI/CD Workflows: GitHub Actions and Release Process

LangChain employs a sophisticated CI/CD system built on GitHub Actions that automates testing, linting, quality checks, and release management across a monorepo structure. The system emphasizes efficiency through intelligent change detection, parallel matrix testing, and strict release gates.

## Architecture Overview

The CI/CD system consists of three layers:

1. **Pull request / push CI (`check_diffs.yml`)**: Detects changed packages and runs targeted tests, linting, and compatibility checks
2. **Scheduled integration testing (`integration_tests.yml`)**: Daily remote API testing with live credentials against partner libraries
3. **Manual release workflow (`_release.yml`)**: Comprehensive pre-release validation, PyPI publishing, and dependent package testing

## Primary CI Workflow (Pull Requests & Master Pushes)

The main entry point is `.github/workflows/check_diffs.yml`, which runs on every pull request, push to master, and merge group event.

### Change Detection & Matrix Generation

The workflow begins with a change detection phase:

1. A Python script (`.github/scripts/check_diff.py`) analyzes which files changed
2. Maps changes to package directories (`libs/core`, `libs/partners/*`, etc.)
3. Builds a dependency graph to include dependent packages when core components change
4. Generates separate test matrices for linting, unit tests, Pydantic compatibility tests, integration test compilation, VCR cassette tests, and extended test suites
5. Outputs are passed as JSON to downstream jobs via matrix strategy

This detection ensures only affected packages are tested, optimizing CI runtime. The script skips the `libs/standard-tests` directory in enumeration and treats certain partners (e.g., huggingface) as CI-unstable, removing them from dependent chains while allowing direct edits to be tested.

### Linting Pipeline (`_lint.yml`)

Runs on affected packages with Python 3.11 (configurable):

- **Ruff analysis**: Code style, import sorting, and rule enforcement with inline GitHub annotations (via `RUFF_OUTPUT_FORMAT: github`)
- **MyPy type checking**: Static type verification
- **Markdown linting**: Documentation quality checks (via `.markdownlint.json`)

Tools are sourced from dependency groups: `lint` and `typing`. The workflow installs both package code and test code dependencies, running `make lint_package` and `make lint_tests` targets. Partner packages receive separate test dependency installation including integration test dependencies.

### Unit Testing (`_test.yml`)

Runs matrix tests across Python versions with dependency constraint verification:

**Matrix dimensions**:
- Python 3.10, 3.11, 3.12, 3.13, 3.14 for `libs/core`
- Python 3.10 and 3.14 for other packages
- Current locked dependencies (from `uv.lock`)
- Minimum supported dependency versions

**Two-phase testing**:

1. **Current dependencies**: Runs full test suite against versions in `uv.lock` via `make test PYTEST_EXTRA=-q`
2. **Minimum dependencies**: Calculates minimum versions from `pyproject.toml` constraints via `get_min_versions.py` script, downgrades via pip, and reruns tests with `make tests PYTEST_EXTRA=-q` to ensure compatibility

The workflow verifies the working directory remains clean (no untracked generated files) after testing.

### Pydantic Compatibility Testing (`_test_pydantic.yml`)

Tests affected packages against configurable Pydantic versions (e.g., v2.0, v2.1, v2.2):

- Triggered when Pydantic version constraints or dependent code changes
- Determines test matrix by querying `uv.lock` for max Pydantic version and `pyproject.toml` for min, across both core and target packages
- Runs `make test` against each Pydantic version
- Uses Python 3.12 by default with override support

### VCR Cassette Tests (`_test_vcr.yml`)

Validates integration tests backed by recorded HTTP cassettes:

- Runs in playback-only mode with fake credentials (no real API keys required)
- Detects stale cassettes from test input changes without re-recording
- Executes `make test_vcr` target
- Enables fast, repeatable integration test feedback

Only triggered for packages with VCR cassettes (currently `libs/partners/openai`), as tracked in the `VCR_PACKAGES` set within `check_diff.py`.

### Integration Test Compilation (`_compile_integration_test.yml`)

Performs shallow integration test validation:

- Compiles test modules without executing them via `pytest -m compile tests/integration_tests`
- Catches import errors and obvious syntax issues
- Provides quick feedback loop without running expensive external API calls
- Installs both test and integration test dependency groups

### Extended Test Suites

For packages defining `extended_testing_deps.txt`, runs additional tests:

- Installs extra dependencies beyond standard test group via the file
- Executes `make extended_tests` target
- Allows performance benchmarks, stress tests, or heavy-weight validations
- Located in the `extended-tests` job within `check_diffs.yml`

### Release Option Validation

The workflow includes a `check-release-options` job:

- Verifies `.github/workflows/_release.yml` dropdown options stay synchronized with actual package directories
- Prevents stale release options from blocking valid releases

## Release Workflow (`_release.yml`)

The release workflow is manually triggered via GitHub Actions UI (or can be called as a reusable workflow). It handles versioning, building, testing, and publishing to PyPI.

### Release Modes & Invocation

**Manual dispatch** (`workflow_dispatch`):
- Dropdown selection of package to release (core, langchain, langchain_v1, text-splitters, standard-tests, model-profiles, or 17+ partner packages)
- Manual version entry (default `0.1.0`)
- Optional override to full path (e.g., `libs/partners/partner-xyz`)
- Dangerous flags: `dangerous-nonmaster-release` (hotfixes), `allow-prereleases`, `skip-prior-published-package-checks`

**Reusable workflow** (`workflow_call`):
- Accepts `working-directory`, `release-version`, and safety bypass flags
- Used internally for multi-package release orchestration

### Release Gate: Build & Version Check

**Job: `build`** (isolated permissions for security):

1. **Version verification**: Extracts version from `pyproject.toml` and compares against input using PEP 440 normalization (treating `0.1.0-rc1` and `0.1.0rc1` as equivalent); fails if mismatch
2. **PyPI availability check**: Queries `https://pypi.org/pypi/{pkg}/{version}/json` to ensure version not already published; fails closed if PyPI is unreachable or returns unexpected status
3. **Build**: Runs `uv build` to create wheel and sdist distributions
4. **Artifact upload**: Stores `dist/` directory for downstream jobs

Security rationale: Separates build (no credentials) from publishing (trusted publishing token) to prevent compromised dependencies from accessing PyPI credentials.

### Release Notes Generation

**Job: `release-notes`**:

1. **Tag detection**: Finds previous release tag via git history
   - For pre-releases (contains hyphen): Matches base version; falls back to latest release tag
   - For stable releases: Searches for previous patch version; falls back to latest
   - First release: Uses full commit history from git root
2. **Changelog extraction**: Runs `git log --format="%s" <prev-tag>..HEAD -- <working-dir>` to collect commit messages
3. **Tag validation**: Confirms previous tag exists in git repo before proceeding

### Pre-Release Checks

**Job: `pre-release-checks`** (no caching to catch missing dependencies):

1. **Direct wheel installation**: Installs built wheel directly via `uv pip install dist/*.whl` (validates metadata and installability)
2. **Package import test**: Verifies main module imports successfully
3. **Unit tests**: Runs full `make tests` against the wheel
4. **Minimum version testing**: Recalculates minimum versions, downgrades via pip, and reruns tests with `make tests PYTEST_EXTRA="-q -k 'not test_serdes'"` (skips serialization tests for speed)
5. **Prerelease dependency detection**: Fails if any dependencies declare prerelease versions (unless release itself is prerelease)
6. **Integration tests**: For partner packages only, runs `make integration_tests` with live API credentials

### PyPI Publishing

**Job: `test-pypi-publish`** (TestPyPI):
- Uses GitHub OpenID Connect (trusted publishing)
- Publishes to test.pypi.org for staging validation
- Tolerates duplicate versions via `skip-existing: true` (CI safety only)

**Job: `publish`** (Production PyPI):
- Uses trusted publishing to production PyPI
- Only runs if all prior checks pass
- Creates GitHub Release with generated release notes

### Compatibility Testing

**Job: `test-prior-published-packages-against-new-core`**:
- Only runs for `libs/core` releases
- Tests previously-published partner packages (currently anthropic, openai) against new core
- Fetches latest non-yanked published partner tag from git, installs new core wheel, runs tests
- Can skip per-partner via `skip-prior-published-package-checks` input (options: none, anthropic, openai, all)

**Job: `test-dependents`**:
- Only runs for `libs/core` or `libs/langchain_v1` releases
- Checks external dependent packages (currently deepagents)
- Tests Python 3.11 and 3.13
- Ensures breaking changes are caught before publish

## Integration Testing (`integration_tests.yml`)

Scheduled daily (1 PM UTC) with manual dispatch override capability.

### Test Matrix Generation

**Job: `compute-matrix`**:

- **Default scope**: Tests 9 partner libraries (OpenAI, Anthropic, Fireworks, Groq, MistralAI, XAI, Google VertexAI, Google GenAI, AWS)
- **Python versions**: 3.10 and 3.14 by default; overridable via input
- **Selective testing**: Can select single library, exclude libraries, or override Python versions
- **Scope security**: Only runs on main repository; manual dispatch allowed from forks

### Integration Test Execution

**Job: `integration-tests`**:

- Checks out primary monorepo plus external `langchain-google` (containing genai and vertexai) and `langchain-aws` repositories
- Reorganizes external repos into local partner directories (`libs/partners/google-genai`, `libs/partners/google-vertexai`, `libs/partners/aws`) for unified testing
- Authenticates to Google Cloud and AWS via respective GitHub Actions
- Runs per-package `make integration_tests` with all live API credentials injected as environment variables
- Uses concurrency locks per (package, python-version) to serialize same-package runs and prevent credential conflicts
- Installs dependencies via `uv sync --group test --group test_integration` for each package
- Includes special per-package install logic: overlays local editable core and standard-tests packages atop checked-out partner versions (e.g., google-genai installs local core and standard-tests; google-vertexai adds langchain_v1; aws adds langchain, anthropic)

**Credentials**: Receives 40+ environment variables covering OpenAI, Anthropic, Google, AWS, Azure, Groq, MistralAI, Deepseek, Cohere, HuggingFace, Together, Mistral, XAI, Perplexity, Upstage, Nvidia, Ollama, OpenRouter, Typesafe, Nomic, MongoDB, Elasticsearch, and LangSmith gateway/tracing.

### External Repository Testing (`test-dependents` Job)

Tests external packages (currently `deepagents`) against local branch versions:

- Checks out external dependent repositories separately
- Installs package with test dependencies via `uv sync --group test`
- Overlays local core/langchain_v1 packages via `uv pip install -e` to test current branch code
- Requires Python >= 3.11, uses bounded matrix from compute-matrix output
- Ensures breaking changes caught before core releases

## Local Development: Pre-Commit Hooks (`.pre-commit-config.yaml`)

The monorepo uses pre-commit hooks to enforce code quality, formatting, and version consistency before commits reach CI. Hooks are automatically triggered on `git commit` after running `pre-commit install`.

### Hook Categories

**Standard validation** (via `pre-commit-hooks`):
- YAML/TOML syntax validation
- File ending verification (ensure newline at EOF)
- Trailing whitespace removal
- Protection against direct commits to `master` branch

**Text normalization** (via `texthooks`):
- Replace curly quotes with straight quotes (`fix-smartquotes`)
- Replace non-standard spaces with regular spaces (`fix-spaces`)

**Per-package format and lint** (via local hooks):
- Each library directory (core, langchain, standard-tests, text-splitters, all partners) has a `format` and `lint` entry
- Triggers `make -C libs/<package> format lint` when files in that package change
- Enforces Ruff formatting and linting rules before commit
- Applied before CI runs, preventing rejected PRs

**Version consistency checks** (via local hooks):
- Ensures `pyproject.toml` version matches source code constants for core, langchain_v1, and all partner packages
- Triggers `make -C libs/<package> check_version` when `pyproject.toml` or version files change
- Prevents version mismatches that would fail CI

### Running Pre-Commit Manually

```bash
# Install hooks
pre-commit install

# Run all hooks on staged files (automatic before commit)
git commit

# Or manually trigger on specific files
pre-commit run --all-files
pre-commit run <hook-id> --all-files
```

## Auto-Labeling Workflows

### Issue Auto-Labeling (`auto-label-by-package.yml`)

Fires when issues are opened or edited:

1. Parses issue body for `## Package` section
2. Supports both dropdown (single select) and checkbox (multi-select) formats
3. Maps package name (e.g., "langchain-openai") to label via JSON mapping table (e.g., "openai")
4. Adds/removes labels to match selected package(s)

### PR Title Linting (`pr_lint.yml`)

Enforces Conventional Commits 1.0.0 format on all pull request titles:

- **Format**: `<type>[optional scope]: <description>` (e.g., `feat(core): add multi-tenant support`)
- **Allowed types**: feat, fix, docs, style, refactor, perf, test, build, ci, chore, revert, release, hotfix
- **Optional scope**: Scopes for specific packages (core, langchain, anthropic, openai, etc.) or cross-cutting concerns (infra, deps, partners)
- **Breaking changes**: Append `!` after type/scope (e.g., `feat!: remove deprecated API`)
- **Release commits**: Must be `release(scope): x.y.z` format
- **Validation**: Uses `amannn/action-semantic-pull-request` with empty scope rejection

Empty scope parentheses are rejected; PR must either omit parentheses (no scope) or provide a valid scope.

### PR Labeling (`pr_labeler.yml`)

Unified PR labeler applying size, file-based, title-based, and contributor classification:

- File-based labels: Maps changed file paths to package labels
- Size labels: Computes PR size (small, medium, large) from diff statistics
- Title-based labels: Detects certain patterns in PR title
- Contributor classification: Checks org membership to tag external contributions (via GitHub App token)
- Uses concurrency locks to prevent race conditions
- Consolidates multiple prior workflows into single sequential run

## OpenWiki Auto-Update (`openwiki-update.yml`)

Runs on schedule (8 AM UTC daily) or manual dispatch:

1. Checks out full repository history via `fetch-depth: 0` (required for diff-against-HEAD)
2. Installs Node.js and OpenWiki CLI (@0.5.0) with optional Mermaid diagram validation
3. Runs `openwiki code --update --print` to regenerate documentation
4. Removes transient state file (`.run.json`)
5. Creates/updates pull request with changes via `peter-evans/create-pull-request@v8.1.1`
6. Preserves partial progress on failure: if OpenWiki run fails, the PR intentionally preserves only pages completed before the failure, allowing them to become baseline for the next scheduled run

Uses LangSmith tracing for observability (OPENWIKI_LANGSMITH_API_KEY, LANGSMITH_API_KEY).

## Dependency Pinning & Version Management

### Frozen Dependency Locks

All CI jobs set `UV_FROZEN=true` and `UV_NO_SYNC=true` (when applicable):

- Ensures reproducible builds against locked versions in `uv.lock`
- Prevents transitive dependency surprises in CI
- Each job explicitly pins Python version and dependency revisions

### Minimum Version Testing

The `get_min_versions.py` script extracts version constraints from `pyproject.toml` and queries PyPI for minimum published versions satisfying those constraints.

Example: If constraint is `langchain-core>=0.3.0,<1.0`, the script finds and installs the earliest 0.3.* release.

Two modes:
- `pull_request`: Tests against minimum with some leniency (used in PR CI)
- `release`: Stricter testing with prerelease rejection (used in release validation)

## Release Policy

### Semantic Versioning

**Core** (`libs/core`) follows strict semantic versioning:
- Major version: Breaking changes
- Minor version: New features (backward compatible)
- Patch version: Bug fixes

**Partner packages** and other libraries align with core releases:
- LangChain follows core versioning for tight integration
- Partners maintain independent versioning but coordinate with core releases

### Release Branching

- Releases only proceed from `master` branch (default) or explicitly via `dangerous-nonmaster-release` flag (hotfixes only)
- Version must match `pyproject.toml` or operator provides override
- PyPI availability double-checked to prevent accidental re-publishes

### Pre-Release Support

- Supports alpha/beta/rc versions (e.g., `0.1.0-rc1`, `0.1.0a1`)
- Pre-release detection normalizes hyphen/underscore variants per PEP 440
- Optional `allow-prereleases` flag permits transitive prerelease dependencies during alpha cycles
- Final releases block prerelease dependencies unless explicitly allowed

## Configuration & Operations

### Environment Variables

**Frozen dependency control** (all CI jobs):
- `UV_FROZEN=true`: Prevents automatic dependency resolution during CI
- `UV_NO_SYNC=true`: Skips uv sync in certain build steps, using manual sync instead

**Linting & formatting** (lint jobs):
- `RUFF_OUTPUT_FORMAT=github`: Inline GitHub annotations for linter violations

**LangSmith tracing** (integration tests):
- `LANGSMITH_TRACING=true`: Enable trace collection
- `LANGSMITH_API_KEY`: API credentials (from `environment: "Scheduled testing"`)
- `LANGSMITH_PROJECT`: Project name (default `scheduled-testing-py`)
- `LANGSMITH_GATEWAY`, `LANGSMITH_GATEWAY_API_KEY`: Optional gateway routing

**Scheduled testing credential scoping**:
- Integration tests run under `environment: "Scheduled testing"` GitHub environment
- Restricts access to 40+ API secrets only to the `integration-tests` and `test-dependents` jobs
- Prevents credential exposure to unrelated jobs
- Required for scheduled runs but unused in PR CI (which can manually override if needed)

### GitHub Actions Permissions

Workflows follow principle of least privilege:

- **Default**: `contents: read` (read-only)
- **PR labeler**: `pull-requests: write`, `issues: write`
- **Release**: `id-token: write` (trusted publishing), `contents: write` (GitHub Release creation)
- **OpenWiki update**: `contents: write`, `pull-requests: write`

Isolated jobs (build, testing) receive no write permissions; publishing jobs run in separate jobs with restricted scope.

### Custom Actions

**`uv_setup`** (`.github/actions/uv_setup`):
- Sets up Python and installs `uv` via `astral-sh/setup-uv@v7`
- Pins `uv` version to `0.12.21` for reproducibility
- Enables optional caching for dependency graphs via cache-dependency-glob:
  - `pyproject.toml` and `uv.lock` (primary)
  - `requirements*.txt` (fallback)
- Supports per-package cache suffixes to avoid cross-contamination
- Parameters: `python-version` (required, MAJOR.MINOR only), `enable-cache` (default true), `cache-suffix`, `working-directory` (default `**`)

## Important Invariants & Failure Modes

1. **No caching in release pre-checks**: Missing dependencies would be masked by cached venvs, allowing broken releases to publish
2. **Minimum version downgrade isolation**: Minimum version tests reinstall packages in fresh virtual environment context, not via constraint relaxation alone
3. **Separate build/publish jobs**: Build job has no PyPI credentials; publishing job has no build tools, preventing supply-chain attacks
4. **Change detection scope**: VCR and extended test matrices only include packages with appropriate markers; adding test files without markers won't trigger corresponding test suites
5. **Prerelease blocking**: Stable releases reject any prerelease dependencies, preventing version resolution issues in downstream users
6. **Tag/version synchronization**: Release workflow validates git tags match expected version format before publishing, catching manual tag drift

## Extension Points

1. **Adding new package types**: Update `check_diff.py` to recognize new directories and map them to appropriate test matrices
2. **Adding partners to release testing**: Update `test-prior-published-packages-against-new-core` matrix and `skip-prior-published-package-checks` input options (keep in sync)
3. **Adding new linting/type checkers**: Extend `_lint.yml` job steps and dependency groups; ensure `make lint_package` target exists
4. **Adding integration test credentials**: Add environment variable to `integration_tests.yml` job and ensure `make integration_tests` target handles optional credentials
5. **Custom test suites**: Create `extended_testing_deps.txt` in package directory and define `make extended_tests` target
6. **OpenWiki pages**: Add to `openwiki/` directory; auto-updated on each scheduled run
