# SPEC: CLI Interface

**Document:** Command-line interface for fileKor  
**Reference:** INIT_SPEC.md section 2

---

## Main Commands

| Command | Description |
|---------|-------------|
| `filekor extract <path>` | Extract text from file |
| `filekor summarize <path>` | Generate summary (short by default) |
| `filekor labels <path>` | Suggest labels |
| `filekor preview <path>` | Generate preview |
| `filekor sidecar <path>` | Generate sidecar JSON file |
| `filekor batch <directory>` | Process complete directory |

---

## Detailed Usage

### `filekor extract`

```bash
filekor extract <path> [OPTIONS]

# Extract text from a PDF
filekor extract documento.pdf

# Extract to specific file
filekor extract documento.pdf --output texto.txt

# Output format
filekor extract documento.pdf --format json
```

**Options:**

| Flag | Description | Default |
|------|-------------|---------|
| `--output, -o` | Output file | stdout |
| `--format, -f` | Format: text, json | text |
| `--cache` | Use cache if exists | False |
| `--no-cache` | Force reprocessing | False |

---

### `filekor summarize`

```bash
filekor summarize <path> [OPTIONS]

# Short summary (default)
filekor summarize documento.pdf

# Long summary
filekor summarize documento.pdf --type long
filekor summarize documento.pdf -t long

# Save to file
filekor summarize documento.pdf --output resumen.txt
```

**Options:**

| Flag | Description | Default |
|------|-------------|---------|
| `--type, -t` | Type: short, long | short |
| `--output, -o` | Output file | stdout |

---

### `filekor labels`

```bash
filekor labels <path> [OPTIONS]

# Suggest labels
filekor labels documento.pdf

# With confidence scores
filekor labels documento.pdf --show-confidence
```

**Example output:**

```
Suggested labels: contract, finance
Confidence:
  contract: 0.92
  finance: 0.88
  provider: 0.30
```

---

### `filekor preview`

```bash
filekor preview <path> [OPTIONS]

# File preview
filekor preview documento.pdf

# Preview with metadata
filekor preview documento.pdf --show-metadata

# Limited lines
filekor preview documento.pdf --lines 50
```

**Example output:**

```
File: contrato-proveedor-2026.pdf
Size: 120 KB | Pages: 12 | Lang: es
----------------------------------------
[Preview]
This service contract establishes the terms and
conditions for cloud services provision...
----------------------------------------
Labels: contract, provider
Summary: Commercial annex with provider pricing...
```

---

### `filekor sidecar`

```bash
filekor sidecar <path> [OPTIONS]

# Generate sidecar
filekor sidecar documento.pdf

# Output directory
filekor sidecar documento.pdf --output ./metadata/

# Force regeneration
filekor sidecar documento.pdf --no-cache
```

**Output:**

Generates `documento.kor` in the same directory or in `--output`.

---

### `filekor batch`

```bash
filekor batch <directory> [OPTIONS]

# Process complete directory
filekor batch ./documentos/

# Analysis level
filekor batch ./documentos/ --level fast

# Only new/modified files
filekor batch ./documentos/ --incremental
```

**Options:**

| Flag | Description | Default |
|------|-------------|---------|
| `--level, -l` | Level: minimal, fast, standard, deep | standard |
| `--output, -o` | Output directory | "." |
| `--incremental` | Only process changes | False |
| `--workers, -w` | Parallel workers | 1 |

---

## Global Flags

| Flag | Description | Default |
|------|-------------|---------|
| `--help, -h` | Show help | - |
| `--version, -v` | Show version | - |
| `--verbose` | Detailed output | False |
| `--quiet, -q` | Suppress output | False |
| `--config` | Config file | ~/.filekor.yaml |
| `--cache-dir` | Cache directory | ~/.filekor/cache |

---

## Config File

```yaml
# ~/.filekor.yaml
filekor:
  version: "1.0"
  
  cache:
    dir: "~/.filekor/cache"
    ttl_days: 30
    
  labels:
    taxonomy: default
    confidence_threshold: 0.7
    
  layers:
    level: standard
    layer_1_threshold: 0.8
    layer_2_threshold: 0.9
    
  llm:
    provider: gemini
    api_key: env:GEMINI_API_KEY
```

---

## Analysis Levels (CLI)

```bash
# Only hash (dup detection)
filekor batch ./docs --level minimal

# Quick metadata
filekor batch ./docs --level fast

# With embeddings
filekor batch ./docs --level standard

# Full LLM
filekor batch ./docs --level deep
```

---

## Usage Examples

```bash
# 1. Scan directory for first time
filekor batch ./proyecto/ --level standard

# 2. Only check new files
filekor batch ./proyecto/ --incremental

# 3. Quick preview of a file
filekor preview ./proyecto/contrato.pdf

# 4. Generate all sidecars
filekor batch ./proyecto/ --output ./proyecto/.filekor/

# 5. Long summary on demand
filekor summarize ./proyecto/doc.pdf --type long
```

---

## Exit Codes

| Code | Meaning |
|------|---------|
| 0 | Success |
| 1 | General error |
| 2 | Argument error |
| 3 | File not found |
| 4 | Corrupted cache |
| 5 | Permission error |