# Publishing studyctl to PyPI and Creating a Homebrew Formula

**Research Date**: 2026-03-15
**Sources**: PyPA official docs, Homebrew docs, uv docs (Context7), Hatch docs, real Homebrew formulae (ansible, jrnl)

---

## Table of Contents

1. [Building from a uv Workspace](#1-building-from-a-uv-workspace)
2. [pyproject.toml for PyPI](#2-pyprojecttoml-for-pypi)
3. [Optional Dependencies / Extras](#3-optional-dependencies--extras)
4. [Homebrew Formula](#4-homebrew-formula)
5. [Homebrew Tap vs homebrew-core](#5-homebrew-tap-vs-homebrew-core)
6. [System Dependencies in Homebrew](#6-system-dependencies-in-homebrew)
7. [Testing Homebrew Formula Locally](#7-testing-homebrew-formula-locally)
8. [uv tool install vs pipx install](#8-uv-tool-install-vs-pipx-install)
9. [Version Management](#9-version-management)
10. [GitHub Actions Workflow](#10-github-actions-workflow)

---

## 1. Building from a uv Workspace

**Question**: Does `uv build` work from a workspace member directory?

**Answer**: Yes. Two approaches:

```bash
# Option A: From workspace root, specify the package
uv build --package studyctl

# Option B: From the member directory
cd packages/studyctl && uv build
```

**Critical recommendation from uv docs**: When publishing, always use `--no-sources`:

```bash
uv build --package studyctl --no-sources
```

**Why**: `--no-sources` disables `tool.uv.sources` resolution. This ensures the package builds correctly outside the workspace (i.e., when installed via pip/uv from PyPI). Without this flag, workspace-local source references could leak into the build, causing failures for end users.

**Output**: Creates `dist/studyctl-X.Y.Z.tar.gz` and `dist/studyctl-X.Y.Z-py3-none-any.whl`.

---

## 2. pyproject.toml for PyPI

### Recommended Changes to `packages/studyctl/pyproject.toml`

```toml
[project]
name = "studyctl"
version = "1.2.0"
description = "AuDHD study pipeline: Obsidian sync, spaced repetition, study plan management"
requires-python = ">=3.12"
readme = "README.md"
license = "MIT"
authors = [{name = "Andy Taylor", email = "andy@netdevautomate.dev"}]
keywords = ["study", "spaced-repetition", "adhd", "education", "cli", "obsidian"]

classifiers = [
    "Development Status :: 4 - Beta",
    "Environment :: Console",
    "Intended Audience :: Education",
    "Intended Audience :: End Users/Desktop",
    "License :: OSI Approved :: MIT License",
    "Operating System :: MacOS",
    "Operating System :: POSIX :: Linux",
    "Programming Language :: Python :: 3",
    "Programming Language :: Python :: 3.12",
    "Programming Language :: Python :: 3.13",
    "Topic :: Education",
    "Typing :: Typed",
]

dependencies = [
    "click>=8.1",
    "rich>=13.0",
    "pyyaml>=6.0",
]

[project.optional-dependencies]
content = ["pymupdf>=1.24", "markitdown>=0.1"]
web = ["fastapi>=0.115", "uvicorn[standard]>=0.30", "jinja2>=3.1"]
notebooklm = ["notebooklm-py>=0.3"]
tui = ["textual>=0.80"]
all = ["studyctl[content,web,notebooklm,tui]"]

[project.scripts]
studyctl = "studyctl.cli:cli"

[project.urls]
Homepage = "https://github.com/NetDevAutomate/Socratic-Study-Mentor"
Repository = "https://github.com/NetDevAutomate/Socratic-Study-Mentor"
Documentation = "https://github.com/NetDevAutomate/Socratic-Study-Mentor#readme"
Issues = "https://github.com/NetDevAutomate/Socratic-Study-Mentor/issues"
Changelog = "https://github.com/NetDevAutomate/Socratic-Study-Mentor/blob/main/CHANGELOG.md"

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.hatch.build.targets.wheel]
packages = ["src/studyctl"]

[tool.hatch.build.targets.sdist]
include = ["src/studyctl", "README.md", "LICENSE"]
```

### Key Points

- **PEP 639** (Dec 2024): `license = "MIT"` as a string is the new standard. `License::` classifiers are deprecated. Hatchling supports this natively.
- **readme**: Must be `README.md` in the **member** directory (`packages/studyctl/README.md`), not the workspace root. This is what PyPI displays.
- **project-urls**: Include `Changelog` -- PyPI renders this as a sidebar link. `Documentation` appears prominently.
- **classifiers**: Use `Development Status :: 4 - Beta` until stable. PyPI rejects uploads with `Private :: Do Not Upload`.
- **keywords**: Used by PyPI search. Keep concise and relevant.

---

## 3. Optional Dependencies / Extras

### Structure Pattern

```toml
[project.optional-dependencies]
content = ["pymupdf>=1.24", "markitdown>=0.1"]
web = ["fastapi>=0.115", "uvicorn[standard]>=0.30", "jinja2>=3.1"]
notebooklm = ["notebooklm-py>=0.3"]
tui = ["textual>=0.80"]
# Meta-extra: installs everything
all = ["studyctl[content,web,notebooklm,tui]"]
```

### How Users Install

```bash
# Core only (click + rich + pyyaml)
pip install studyctl

# With web server
pip install studyctl[web]

# With TUI and content processing
pip install studyctl[tui,content]

# Everything
pip install studyctl[all]

# uv equivalents
uv tool install studyctl
uv tool install 'studyctl[web]'
uv tool install 'studyctl[all]'
```

### Guarding Optional Imports in Code

```python
# studyctl/web/server.py
def start_server():
    try:
        import fastapi
    except ImportError:
        import click
        raise click.UsageError(
            "Web features require the 'web' extra.\n"
            "Install with: pip install studyctl[web]"
        )
    # ... proceed with fastapi
```

Or using a lazy-import pattern with a decorator:

```python
import functools
import click

def requires_extra(extra_name: str, package: str):
    """Decorator that checks for optional dependency availability."""
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            try:
                __import__(package)
            except ImportError:
                raise click.UsageError(
                    f"This command requires the '{extra_name}' extra.\n"
                    f"Install with: pip install studyctl[{extra_name}]"
                )
            return func(*args, **kwargs)
        return wrapper
    return decorator

# Usage:
@click.command()
@requires_extra("web", "fastapi")
def serve():
    """Start the web UI."""
    from studyctl.web.server import run
    run()
```

---

## 4. Homebrew Formula

### Pattern: `Language::Python::Virtualenv`

Homebrew Python apps since Python 3.12 **must** use a virtualenv in `libexec` (PEP 668 compliance). The `Language::Python::Virtualenv` mixin handles this.

### Example Formula for studyctl

Create as `Formula/studyctl.rb` in your tap:

```ruby
class Studyctl < Formula
  include Language::Python::Virtualenv

  desc "AuDHD study pipeline: spaced repetition, Obsidian sync, study plans"
  homepage "https://github.com/NetDevAutomate/Socratic-Study-Mentor"
  url "https://files.pythonhosted.org/packages/.../studyctl-1.2.0.tar.gz"
  sha256 "abc123..."
  license "MIT"

  depends_on "libyaml"
  depends_on "python@3.13"

  # Optional system deps -- inform user via caveats
  uses_from_macos "libffi"

  # All Python dependencies as resources
  # Generated by: brew update-python-resources studyctl
  resource "click" do
    url "https://files.pythonhosted.org/packages/.../click-8.1.8.tar.gz"
    sha256 "..."
  end

  resource "rich" do
    url "https://files.pythonhosted.org/packages/.../rich-13.9.4.tar.gz"
    sha256 "..."
  end

  resource "pyyaml" do
    url "https://files.pythonhosted.org/packages/.../pyyaml-6.0.2.tar.gz"
    sha256 "..."
  end

  resource "markdown-it-py" do
    url "https://files.pythonhosted.org/packages/.../markdown_it_py-3.0.0.tar.gz"
    sha256 "..."
  end

  resource "mdurl" do
    url "https://files.pythonhosted.org/packages/.../mdurl-0.1.2.tar.gz"
    sha256 "..."
  end

  resource "pygments" do
    url "https://files.pythonhosted.org/packages/.../pygments-2.19.2.tar.gz"
    sha256 "..."
  end

  def install
    virtualenv_install_with_resources
  end

  test do
    assert_match "Usage", shell_output("#{bin}/studyctl --help")
    assert_match version.to_s, shell_output("#{bin}/studyctl --version")
  end

  def caveats
    <<~EOS
      studyctl has optional features that require additional tools:

        Mermaid diagrams:  brew install mermaid-cli
        PDF export:        brew install pandoc

      For the full-featured version with web UI, TUI, and content processing:
        pip install studyctl[all]
      or:
        uv tool install 'studyctl[all]'
    EOS
  end
end
```

### Key Points

- **`virtualenv_install_with_resources`**: This single method creates a venv in `libexec`, installs all `resource` stanzas into it, installs the package itself, and symlinks executables to `bin`.
- **`brew update-python-resources studyctl`**: Auto-generates all `resource` stanzas from PyPI metadata. Run this after publishing to PyPI.
- **`homebrew-pypi-poet`**: Alternative if `brew update-python-resources` fails.
- **Homebrew formula only installs core deps**. Extras (web, tui, notebooklm, content) are not included -- users who want those use pip/uv directly. Document this in caveats.

### Generating Resource Stanzas

```bash
# After publishing studyctl to PyPI:
brew update-python-resources --print-only studyctl

# Or using homebrew-pypi-poet:
pip install homebrew-pypi-poet
poet studyctl
```

---

## 5. Homebrew Tap vs homebrew-core

### homebrew-core Requirements (All Must Be Met)

| Criterion | Threshold | studyctl Status |
|---|---|---|
| Stable release | Not "beta" or "unstable" | Needs v1.0+ stable |
| Maintainable | Works unpatched on all supported macOS + Linux | Needs verification |
| Known/Notable | >=75 GitHub stars OR >=30 forks OR >=30 watchers | **Not yet met** |
| Used by others | Someone other than author must submit/request | **Not yet met** |
| Homepage | Public webpage/README | Met |

### Recommendation: Start with a Personal Tap

```bash
# Create your tap
brew tap-new NetDevAutomate/homebrew-studyctl

# This creates:
# /opt/homebrew/Library/Taps/NetDevAutomate/homebrew-studyctl/
#   Formula/
#   .github/workflows/  (auto-generated CI for bottle building)

# Push to GitHub
gh repo create NetDevAutomate/homebrew-studyctl --push --public \
  --source "$(brew --repository NetDevAutomate/homebrew-studyctl)"

# Users install via:
brew tap NetDevAutomate/studyctl
brew install studyctl
# Or one-liner:
brew install NetDevAutomate/studyctl/studyctl
```

### Tap Benefits

- **Auto-bottling**: The default `.github/workflows/` from `brew tap-new` builds bottles (precompiled binaries) and uploads to GitHub Releases automatically on PR merge.
- **Full control**: No review process, any package size/popularity.
- **Migration path**: When you hit the thresholds, submit to homebrew-core.

---

## 6. System Dependencies in Homebrew

### Pattern: Required vs Optional System Deps

```ruby
# REQUIRED: Always installed
depends_on "python@3.13"
depends_on "libyaml"

# BUILD-ONLY: Not needed at runtime
depends_on "rust" => :build  # if any dep needs compilation

# FROM MACOS: Use system version if available
uses_from_macos "libffi"

# OPTIONAL: Use caveats, NOT depends_on
# Homebrew doesn't support optional deps well -- use caveats instead
def caveats
  <<~EOS
    For Mermaid diagram support:
      brew install mermaid-cli

    For PDF export:
      brew install pandoc
  EOS
end
```

### Why Not `depends_on` for Optional Deps?

Homebrew formulae don't have a concept of optional Python extras. If you add `depends_on "pandoc"`, **every** user who installs studyctl gets pandoc (200MB+). Instead:

1. Homebrew formula installs **core deps only** (click, rich, pyyaml)
2. Caveats inform users about optional system tools
3. Users who want extras use `pip install studyctl[all]` or `uv tool install 'studyctl[all]'`

### Recommended Approach for pandoc/mmdc

```ruby
def caveats
  s = ""
  unless Formula["pandoc"].any_version_installed?
    s += "For PDF export: brew install pandoc\n"
  end
  unless which("mmdc")
    s += "For Mermaid diagrams: brew install mermaid-cli\n"
  end
  s.empty? ? nil : s
end
```

Actually, the simpler static caveats approach is preferred by Homebrew maintainers -- dynamic caveats are discouraged.

---

## 7. Testing Homebrew Formula Locally

### Step-by-Step Testing Workflow

```bash
# 1. Create your tap (if not done)
brew tap-new NetDevAutomate/homebrew-studyctl

# 2. Create the formula file
$EDITOR "$(brew --repository NetDevAutomate/homebrew-studyctl)/Formula/studyctl.rb"

# 3. Install from local formula (builds from source)
brew install --verbose --debug NetDevAutomate/studyctl/studyctl

# 4. Run the formula's test block
brew test NetDevAutomate/studyctl/studyctl

# 5. Audit for style/correctness
brew audit --strict --online --new NetDevAutomate/studyctl/studyctl

# 6. Check the installed files
brew list NetDevAutomate/studyctl/studyctl

# 7. Verify the binary works
studyctl --help
studyctl --version

# 8. Uninstall and cleanup
brew uninstall NetDevAutomate/studyctl/studyctl
```

### Pre-Submission Checklist

- [ ] `brew install` succeeds from source
- [ ] `brew test` passes
- [ ] `brew audit --strict --online --new` passes with no errors
- [ ] Binary is in PATH and runs correctly
- [ ] `--help` and `--version` produce expected output
- [ ] Formula works on both Apple Silicon and Intel (if targeting homebrew-core)

---

## 8. uv tool install vs pipx install

### Comparison

| Feature | `uv tool install` | `pipx install` |
|---|---|---|
| Speed | 10-100x faster (Rust-based) | Standard pip speed |
| Isolation | Dedicated venv per tool | Dedicated venv per tool |
| Extras support | `uv tool install 'studyctl[web]'` | `pipx install 'studyctl[web]'` |
| Python version | Uses uv-managed Python | Uses system/pyenv Python |
| Binary location | `~/.local/bin` | `~/.local/bin` |
| Upgrade | `uv tool upgrade studyctl` | `pipx upgrade studyctl` |

### Supporting Both in Documentation

Add this to README.md:

```markdown
## Installation

### Recommended: uv (fastest)
```bash
uv tool install studyctl

# With extras:
uv tool install 'studyctl[web,tui]'
```

### Alternative: pipx
```bash
pipx install studyctl

# With extras:
pipx install 'studyctl[web,tui]'
```

### Alternative: pip (in a venv)
```bash
python -m venv .venv && source .venv/bin/activate
pip install studyctl[all]
```

### macOS: Homebrew
```bash
brew tap NetDevAutomate/studyctl
brew install studyctl
```
```

### Key Insight

Both `uv tool install` and `pipx install` work identically from PyPI. No special configuration needed -- publishing to PyPI supports both automatically. The `[project.scripts]` entry point is what both tools use to create the CLI binary.

---

## 9. Version Management

### Recommended: uv version (Single Source of Truth)

Since uv 0.7+, `uv version` manages versions directly in `pyproject.toml`:

```bash
# View current version
uv version --package studyctl

# Set exact version
uv version --package studyctl 1.2.0

# Bump semantically
uv version --package studyctl --bump patch   # 1.2.0 -> 1.2.1
uv version --package studyctl --bump minor   # 1.2.0 -> 1.3.0
uv version --package studyctl --bump major   # 1.2.0 -> 2.0.0

# Preview without changing
uv version --package studyctl --bump minor --dry-run
```

### Alternative: hatch-vcs (Git Tag as Source of Truth)

If you prefer git tags to drive versions:

```toml
# packages/studyctl/pyproject.toml
[project]
name = "studyctl"
dynamic = ["version"]

[tool.hatch.version]
source = "vcs"
raw-options = { root = "../.." }  # point to workspace root where .git lives

[tool.hatch.build.hooks.vcs]
version-file = "src/studyctl/_version.py"
```

**Trade-offs**:

| Approach | Pros | Cons |
|---|---|---|
| `uv version` (static) | Simple, explicit, no plugins needed | Must remember to bump before tagging |
| `hatch-vcs` (dynamic) | Tag = version, no drift possible | Extra plugin dep, `root` path needed for monorepo |

### Recommendation for studyctl

Use **`uv version`** (static approach). Reasons:
1. No additional build dependencies
2. Works naturally with `uv build --package studyctl`
3. Version is visible in `pyproject.toml` (easy to audit)
4. The release workflow tags after bumping, not the other way around

### Workspace Version Bumping

For the workspace root and both members, use a simple script:

```bash
#!/usr/bin/env bash
# scripts/bump-version.sh
set -euo pipefail

VERSION="${1:?Usage: bump-version.sh <version>}"

echo "Bumping studyctl to ${VERSION}..."
uv version --package studyctl "${VERSION}"

echo "Done. Don't forget to:"
echo "  git add packages/studyctl/pyproject.toml"
echo "  git commit -m 'chore: bump studyctl to ${VERSION}'"
echo "  git tag -a v${VERSION} -m 'Release ${VERSION}'"
echo "  git push origin main --tags"
```

---

## 10. GitHub Actions Workflow

### Complete Workflow: Build, Test, Publish to PyPI

```yaml
# .github/workflows/publish-pypi.yml
name: Publish studyctl to PyPI

on:
  push:
    tags:
      - "v*"  # Triggers on v1.0.0, v1.2.3, etc.

permissions:
  contents: read

jobs:
  # ──────────────────────────────────────────────
  # Job 1: Run tests before publishing
  # ──────────────────────────────────────────────
  test:
    name: Test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
        with:
          persist-credentials: false

      - name: Install uv
        uses: astral-sh/setup-uv@v5

      - name: Set up Python
        run: uv python install 3.12

      - name: Install all workspace packages
        run: uv sync --all-packages

      - name: Run tests
        run: uv run pytest packages/studyctl/tests/ -v

      - name: Lint
        run: uv run ruff check packages/studyctl/

  # ──────────────────────────────────────────────
  # Job 2: Build distribution packages
  # ──────────────────────────────────────────────
  build:
    name: Build distribution
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
        with:
          persist-credentials: false

      - name: Install uv
        uses: astral-sh/setup-uv@v5

      - name: Set up Python
        run: uv python install 3.12

      - name: Build studyctl sdist and wheel
        run: uv build --package studyctl --no-sources

      - name: Verify dist contents
        run: |
          ls -la dist/
          # Ensure both sdist and wheel were created
          test -f dist/studyctl-*.tar.gz
          test -f dist/studyctl-*.whl

      - name: Store distribution packages
        uses: actions/upload-artifact@v5
        with:
          name: python-package-distributions
          path: dist/

  # ──────────────────────────────────────────────
  # Job 3: Publish to TestPyPI (on all tags)
  # ──────────────────────────────────────────────
  publish-testpypi:
    name: Publish to TestPyPI
    needs: build
    runs-on: ubuntu-latest
    environment:
      name: testpypi
      url: https://test.pypi.org/p/studyctl
    permissions:
      id-token: write  # Required for trusted publishing
    steps:
      - name: Download distributions
        uses: actions/download-artifact@v6
        with:
          name: python-package-distributions
          path: dist/

      - name: Publish to TestPyPI
        uses: pypa/gh-action-pypi-publish@release/v1
        with:
          repository-url: https://test.pypi.org/legacy/

  # ──────────────────────────────────────────────
  # Job 4: Publish to PyPI (on release tags only)
  # ──────────────────────────────────────────────
  publish-pypi:
    name: Publish to PyPI
    needs: [build, publish-testpypi]
    runs-on: ubuntu-latest
    environment:
      name: pypi
      url: https://pypi.org/p/studyctl
    permissions:
      id-token: write  # Required for trusted publishing
    steps:
      - name: Download distributions
        uses: actions/download-artifact@v6
        with:
          name: python-package-distributions
          path: dist/

      - name: Publish to PyPI
        uses: pypa/gh-action-pypi-publish@release/v1

  # ──────────────────────────────────────────────
  # Job 5: Create GitHub Release
  # ──────────────────────────────────────────────
  github-release:
    name: Create GitHub Release
    needs: publish-pypi
    runs-on: ubuntu-latest
    permissions:
      contents: write  # Required for creating releases
    steps:
      - name: Download distributions
        uses: actions/download-artifact@v6
        with:
          name: python-package-distributions
          path: dist/

      - name: Create GitHub Release
        env:
          GITHUB_TOKEN: ${{ github.token }}
        run: |
          gh release create "${{ github.ref_name }}" dist/* \
            --repo "${{ github.repository }}" \
            --title "studyctl ${{ github.ref_name }}" \
            --generate-notes
```

### Setting Up Trusted Publishing (One-Time)

Trusted Publishing uses OpenID Connect -- no API tokens needed.

1. **Go to**: https://pypi.org/manage/account/publishing/
2. **Add a new pending publisher** (before first publish):
   - PyPI Project Name: `studyctl`
   - Owner: `NetDevAutomate`
   - Repository: `Socratic-Study-Mentor`
   - Workflow name: `publish-pypi.yml`
   - Environment name: `pypi`
3. **Repeat for TestPyPI**: https://test.pypi.org/manage/account/publishing/
   - Same settings but environment name: `testpypi`
4. **Create GitHub environments**: Settings > Environments > New
   - Create `pypi` and `testpypi` environments
   - Optionally add protection rules (require approval for `pypi`)

### Alternative: Using uv publish Directly

If you prefer `uv publish` over the GitHub Action:

```yaml
      - name: Install uv
        uses: astral-sh/setup-uv@v5

      - name: Publish to PyPI
        run: uv publish dist/*
        # uv publish auto-detects trusted publishing in GitHub Actions
```

Both approaches work. The `pypa/gh-action-pypi-publish` action is the PyPA-blessed approach and generates PEP 740 attestations automatically since v1.11.0.

---

## Release Checklist

```
1. [ ] Update CHANGELOG.md
2. [ ] Bump version: ./scripts/bump-version.sh 1.2.0
3. [ ] Verify build: uv build --package studyctl --no-sources
4. [ ] Test install: uv run --with ./dist/studyctl-1.2.0-py3-none-any.whl --no-project -- studyctl --version
5. [ ] Commit: git add -A && git commit -m "chore: release studyctl 1.2.0"
6. [ ] Tag: git tag -a v1.2.0 -m "Release 1.2.0"
7. [ ] Push: git push origin main --tags
8. [ ] Monitor: GitHub Actions publishes to TestPyPI -> PyPI
9. [ ] Verify: pip install --index-url https://test.pypi.org/simple/ studyctl
10. [ ] Update Homebrew tap: brew update-python-resources studyctl
```

---

## Quick Reference: TestPyPI Configuration

For testing before the real release, add to workspace `pyproject.toml`:

```toml
[[tool.uv.index]]
name = "testpypi"
url = "https://test.pypi.org/simple/"
publish-url = "https://test.pypi.org/legacy/"
explicit = true
```

Then:

```bash
# Build
uv build --package studyctl --no-sources

# Publish to TestPyPI manually (for testing)
uv publish --index testpypi --token $TEST_PYPI_TOKEN

# Test install from TestPyPI
uv run --index-url https://test.pypi.org/simple/ \
  --extra-index-url https://pypi.org/simple/ \
  --with studyctl --no-project -- studyctl --help
```

---

## Files to Create Before First Publish

1. **`packages/studyctl/README.md`** -- PyPI landing page (supports Markdown)
2. **`packages/studyctl/LICENSE`** -- MIT license text
3. **`CHANGELOG.md`** -- Release history
4. **`.github/workflows/publish-pypi.yml`** -- CI/CD workflow (above)
5. **`scripts/bump-version.sh`** -- Version bumping helper
6. **Homebrew tap repo** -- `NetDevAutomate/homebrew-studyctl` on GitHub
