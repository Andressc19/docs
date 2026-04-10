# fileKor — Initial Specification

**Version:** 1.0  
**Date:** April 2026  
**Author:** Andres Camperos  
**Document Type:** Project Specification  
**Status:** Pre-Development (Phase 0)

---

## 1. Product Definition

**fileKor** is a local metadata engine that extracts, summarizes, classifies, and tags files — consumed as a library (PyPI) or used as a CLI.

| Mode | Description |
|------|-------------|
| **PyPI** | Imported as a Python module |
| **CLI** | Executed from terminal |

**fileKor is NOT:** Complete TUI, search engine, centralized indexing system, cloud service.

---

## 2. API Contract

### Main Interface

```python
class MetadataEngine(ABC):
    def extract_text(self, path: Path) -> ExtractedText: pass
    def summarize(self, path: Path, summary_type: SummaryType) -> SummaryResult: pass
    def suggest_labels(self, path: Path) -> LabelSuggestion: pass
    def get_preview(self, path: Path) -> PreviewResult: pass
    def generate_sidecar(self, path: Path) -> SidecarFile: pass
```

### Main DTOs

```python
@dataclass
class ExtractedText:
    text_content: str
    page_count: int
    word_count: int
    language: str
    encoding: str

@dataclass
class SummaryResult:
    summary: str
    summary_type: SummaryType  # SHORT or LONG
    key_points: List[str]
    entities: List[str]
    labels: List[str]
    generated_at: datetime

@dataclass
class LabelSuggestion:
    suggested_labels: List[str]
    confidence_scores: Dict[str, float]
    reasoning: str

@dataclass
class PreviewResult:
    content_snippet: str
    metadata: dict
    mime_type: str
    line_count: int
    short_summary: str

@dataclass
class SidecarFile:
    version: str = "1.0"
    file: FileInfo
    metadata: Optional[MetadataInfo] = None
    content: Optional[ContentInfo] = None
    summary: Optional[SummaryInfo] = None
    labels: Optional[LabelsInfo] = None
    parser_status: ParserStatus
    generated_at: datetime
    generated_by: str = "filekor"
```

> Detailed JSON schema: see [SPEC_SIDECAR.md](SPEC_SIDECAR.md)

```python
# Internal DTOs for SidecarFile
@dataclass
class FileInfo:
    path: str
    name: str
    extension: str
    size_bytes: int
    modified_at: str  # ISO 8601
    hash_sha256: str

@dataclass
class MetadataInfo:
    author: Optional[str] = None
    created: Optional[str] = None
    pages: Optional[int] = None

@dataclass
class ContentInfo:
    language: Optional[str] = None
    word_count: Optional[int] = None

@dataclass
class SummaryInfo:
    short: Optional[str] = None
    long: Optional[str] = None

@dataclass
class LabelsInfo:
    suggested: List[str] = None
    confidence: Dict[str, float] = None
```

### Enums

```python
class SummaryType(Enum):
    SHORT = "short"
    LONG = "long"

class ParserStatus(Enum):
    OK = "OK"
    DEGRADED = "DEGRADED"
    BROKEN = "BROKEN"
```

---

## 3. Architecture

### Adapter Pattern

```
fileKor Core
        ↓
   Adapter (common interface)
        ↓
External Tool (pdfplumber, PyExifTool, embeddings, LLM)
```

### Technology Stack

| Package | Role | Version | Python Support | License |
|---------|------|---------|----------------|---------|
| python | Runtime | ^3.11 | - | - |
| pdfplumber | PDF text extraction | ^0.11.0 | >=3.8 | MIT |
| PyExifTool | Metadata extraction | ^0.6.0 | >=3.7 | GPL/BSD |
| click | CLI framework | ^8.1.0 | >=3.7 | BSD-3-Clause |
| rich | Formatted output | ^13.0.0 | >=3.7 | MIT |
| pydantic | DTO validation | ^2.0.0 | >=3.7 | MIT |
| pyyaml | Config parsing | ^6.0 | >=3.7 | MIT |

---

## 4. Layer Model

See full document: [SPEC_LAYERS.md](SPEC_LAYERS.md)

| Layer | Name | Description | Cost |
|-------|------|-------------|------|
| **L0** | Hash Dedup | SHA256 cache check | $0 |
| **L1** | Native + OS | Native metadata (exiftool) | $0 |
| **L2** | Embeddings | Similarity check (adapter pattern) | $0 |
| **L3** | LLM Prompt | Gemini + Ollama (adapter pattern) | $ |

### Analysis Levels

| Level | Layers | Cost | Precision |
|-------|--------|------|-----------|
| **minimal** | L0 | $0 | Low |
| **fast** | L0 + L1 | $0 | Medium |
| **standard** | L0 + L1 + L2 | $0 | High |
| **deep** | All | $ | Very High |

### Confidence Threshold

- Hardcoded in Phase 0 (e.g: 0.8)
- Configurable via `config.yaml` in later phases

---

## 5. Sidecar Schema

See full document: [SPEC_SIDECAR.md](SPEC_SIDECAR.md)

---

## 6. Label Taxonomy

See full document: [SPEC_TAXONOMY.md](SPEC_TAXONOMY.md)

- **Predefined base:** `contract, finance, legal, provider, architecture, config, notes`
- **Configurable:** Via consumer file

---

## 7. Error Handling

| Parser Status | Meaning |
|---------------|---------|
| `OK` | Extracted correctly |
| `DEGRADED` | Encrypted or partial — `"Needs OCR or password"` |
| `BROKEN` | Cannot be read |

**Philosophy:** Don't attempt to decrypt, mark and continue.

---

## 8. Construction by Phases

See full document: [SPEC_PHASES.md](SPEC_PHASES.md)

### Phase 0 — Fundamentals (Testable)
- Layer 0: Hash + cache (stdlib)
- Layer 1: Adapter + Mock
- Unit tests

### Phase 1 — Real Metadata
- Real PyExifTool
- Sidecar generation
- Incremental cache

### Phase 2 — Embeddings
- EmbeddingsAdapter interface
- Mock + real implementation

### Phase 3 — LLM
- LLMAdapter interface
- Gemini + Ollama implementations

---

## 9. Development Principles

- **Mock-first:** Build against interfaces
- **Adapter pattern:** Don't couple to specific tools
- **Testable:** Each layer testable independently
- **Local-only:** No cloud services
- **Layered construction:** Don't add complexity prematurely

---

## 10. References

| Document | Purpose |
|----------|---------|
| [SPEC_LAYERS.md](SPEC_LAYERS.md) | Detailed layer model |
| [SPEC_SIDECAR.md](SPEC_SIDECAR.md) | Complete sidecar schema |
| [SPEC_TAXONOMY.md](SPEC_TAXONOMY.md) | Label taxonomy |
| [SPEC_PHASES.md](SPEC_PHASES.md) | Phase-based construction plan |
| [SPEC_CLI.md](SPEC_CLI.md) | Detailed CLI interface |
| questions.md | Answered design decisions |

---

## 11. Decisions Made

| # | Decision |
|---|----------|
| D001 | pdfplumber for text extraction |
| D002 | PyExifTool for metadata |
| D003 | Hybrid taxonomy (predefined + configurable) |
| D004 | Sidecars + cache for persistence |
| D005 | Incremental extraction with hash comparison |
| D006 | Encrypted files → DEGRADED |
| D007 | Layered model for summaries |
| D008 | CLI + PyPI as consumption modes |
| D009 | Layer 2 uses Adapter pattern (any library) |
| D010 | Layer 3 uses Gemini + Ollama (Adapter pattern) |
| D011 | Confidence threshold hardcoded + configurable |
| D012 | Sidecar schema derives from consumer's FileRecord |
| D013 | Metadata files named `.kor` (e.g: `file.kor`) |
