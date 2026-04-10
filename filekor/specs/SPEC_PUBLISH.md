# SPEC: Package Publication

**Document:** Python package configuration and PyPI publication strategy  
**Reference:** INIT_SPEC.md section 2, D008 (CLI + PyPI as consumption modes)

---

## 1. Package Manager

**setuptools** is the build backend for fileKor.

Standard Python packaging tool that works well with uv, pip, and Poetry. Supports editable installs and src layout out of the box.

---

## 2. pyproject.toml Configuration

```toml
[build-system]
requires = ["setuptools>=61.0"]
build-backend = "setuptools.build_meta"

[project]
name = "filekor"
version = "0.1.0"
description = "Local metadata engine that extracts, summarizes, classifies, and tags files"
authors = [{name = "Andres Camperos", email = "andres@email.com"}]
readme = "README.md"
license = {text = "MIT"}
requires-python = ">=3.11"

classifiers = [
    "Development Status :: 3 - Alpha",
    "Intended Audience :: Developers",
    "License :: OSI Approved :: MIT License",
    "Programming Language :: Python :: 3",
    "Programming Language :: Python :: 3.11",
    "Programming Language :: Python :: 3.12",
]

dependencies = [
    "click>=8.1.0",
    "rich>=13.0.0",
    "pydantic>=2.0.0",
    "pyyaml>=6.0",
]

[project.optional-dependencies]
dev = ["pytest", "pytest-cov", "black", "ruff", "mypy"]

[project.scripts]
filekor = "filekor.cli:main"

[tool.setuptools.packages.find]
where = ["src"]

[tool.setuptools.package-dir]
filekor = "src/filekor"
```

---

## 3. Dependencies Strategy

### Core Dependencies (Required)

| Package | Version | Purpose |
|---------|---------|---------|
| click | ^8.1.0 | CLI framework |
| rich | ^13.0.0 | Formatted terminal output |
| pydantic | ^2.0.0 | DTO validation |
| pyyaml | ^6.0 | Config file parsing |

### Optional Dependencies (Phase-based)

| Package | Phase | Purpose |
|---------|-------|---------|
| PyExifTool | Phase 1 | Real metadata extraction |
| sentence-transformers | Phase 2 | Embeddings (enables L2) |
| google-generativeai | Phase 3 | Gemini API integration |
| ollama | Phase 3 | Local LLM (enables L3) |

### Installation Modes

```bash
pip install filekor              # Core only (MVP)
pip install filekor[dev]         # With development tools
pip install filekor[full]        # With all optional dependencies
```

---

## 4. Version Strategy

### Semantic Versioning (SemVer)

```
MAJOR.MINOR.PATCH
  │    │    │
  │    │    └── Bug fixes, no API changes
  │    └────── New features, backward compatible
  └─────────── Breaking changes
```

### Version Progression

| Phase | Version | Rationale |
|-------|---------|-----------|
| Phase 0-1 | 0.1.0 - 0.x.y | Alpha - MVP, unstable API |
| Phase 2 | 1.0.0-beta | Feature complete, testing |
| Phase 3+ | 1.0.0 | Stable release |

### Version Bumping

```bash
# Manual version bump in pyproject.toml
# Then build and publish
python -m build
twine upload dist/*
```

---

## 5. CLI Entry Point

The `[tool.poetry.scripts]` section in pyproject.toml creates a `filekor` command:

```toml
[tool.poetry.scripts]
filekor = "filekor.cli:main"
```

After installation:

```bash
filekor --version
filekor extract documento.pdf
python -m filekor extract documento.pdf
```

---

## 6. Publication Process

### Manual Publication

```bash
# Update version
poetry version patch

# Build package
poetry build

# Publish to PyPI
poetry publish --username __token__ --password $PYPI_TOKEN
```

### Automated Publication (Future)

GitHub Actions workflow for automatic publishing on release:

```yaml
# .github/workflows/publish.yml
name: Publish to PyPI

on:
  release:
    types: [created]

jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Install Poetry
        run: pip install poetry
      - name: Install dependencies
        run: poetry install
      - name: Build package
        run: poetry build
      - name: Publish to PyPI
        run: poetry publish --username __token__ --password ${{ secrets.PYPI_TOKEN }}
```

---

## 7. Development Workflow

### Virtual Environment (uv)

**uv** is the recommended tool for managing virtual environments. It is significantly faster than pip and Poetry for dependency installation.

```bash
# Create virtual environment with uv
uv venv
source .venv/bin/activate

# Install project in editable mode with dev dependencies
uv pip install -e ".[dev]"

# Install all optional dependencies
uv pip install -e ".[full]"
```

### Testing Before Publication

```bash
# Build package locally
python -m build

# Verify package contents
tar -tzf dist/filekor-0.1.0.tar.gz

# Install locally for testing
pip install dist/filekor-0.1.0.tar.gz
```

### Local Installation

```bash
poetry install                    # Editable mode (development)
poetry install --with dev         # With dev dependencies
poetry install --all-extras      # All optional dependencies
```

### Testing Before Publication

```bash
poetry build                                    # Build locally
tar -tzf dist/filekor-0.1.0.tar.gz              # Verify contents
pip install dist/filekor-0.1.0.tar.gz           # Install locally
```

---

## 8. Package Validation

### Pre-publication Checklist

- [ ] Version updated in pyproject.toml
- [ ] README.md contains accurate description
- [ ] LICENSE file present
- [ ] All dependencies declared
- [ ] CLI entry point tested locally
- [ ] Package builds without errors
- [ ] Package installs correctly in virtual environment
- [ ] Tests pass

### Validation Commands

```bash
poetry show                              # Check metadata
poetry build && twine check dist/*       # Build and verify
poetry publish --dry-run                 # Dry-run
```

---

## 9. Distribution Formats

### Wheel (Primary)

Faster installation, pre-built, no build step on user machine. Format: `filekor-0.1.0-py3-none-any.whl`

### Source Distribution (Secondary)

Required for edge cases. Format: `filekor-0.1.0.tar.gz`

Poetry generates both formats with `poetry build`.

---

## 10. Environment Variables for CI/CD

| Variable | Purpose |
|----------|---------|
| `PYPI_TOKEN` | PyPI API token for publication |
| `TEST_PYPI_TOKEN` | Test PyPI token for testing |

Tokens are stored in GitHub Secrets or equivalent secrets management. Never commit tokens to repository.

---

## 11. Timeline

| Milestone | Version | Trigger |
|-----------|---------|---------|
| MVP Release | 0.1.0 | Phase 0-1 complete |
| First Feature Release | 0.2.0 | Phase 2 (embeddings) |
| Stable Release | 1.0.0 | Phase 3 (LLM) complete |

---

## 12. Related Decisions

| Decision | Reference |
|----------|-----------|
| CLI as consumption mode | INIT_SPEC.md D008 |
| Adapter pattern for tools | INIT_SPEC.md section 3 |
| Layered construction | SPEC_PHASES.md |

---

## 13. Project Structure

### src Layout

fileKor uses the modern **src layout** to avoid `fileKor/filekor/` directory duplication:

```
fileKor/
├── pyproject.toml
├── src/
│   └── filekor/
│       ├── __init__.py
│       ├── core.py
│       ├── cli.py
│       └── adapters/
│           ├── __init__.py
│           ├── base.py
│           ├── mock.py
│           └── real.py
└── tests/
    ├── unit/
    └── integration/
```

### Rationale

- Avoids `fileKor/filekor/` directory structure
- Clear separation between project config and source code
- Standard modern Python practice
- Works with uv, poetry, and pip

### Installation

```bash
uv venv
source .venv/bin/activate
uv pip install -e ".[dev]"  # Works with src layout
```

---

## Summary

| Aspect | Decision |
|--------|----------|
| Build Backend | setuptools |
| Package Manager | uv (for dev) |
| Publication Target | PyPI |
| Version Strategy | SemVer (0.x.y for MVP) |
| CLI Installation | Automatic via entry point |
| CI/CD | GitHub Actions (future) |