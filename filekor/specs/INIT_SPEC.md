# fileKor — Initial Specification

**Version:** 1.0  
**Date:** April 2026  
**Author:** Andres Camperos
**Document Type:** Project Specification  
**Status:** Pre-Development (Phase 0)

---

## 1. Product Definition

**fileKor** es un motor de metadatos local que extrae, resume, clasifica y etiqueta archivos — consumido como librería (PyPI) o usado como CLI.

| Modo | Descripción |
|------|-------------|
| **PyPI** | Se importa como módulo Python |
| **CLI** | Se ejecuta desde terminal |

**fileKor NO es:** TUI completa, motor de búsqueda, sistema de indexación central, servicio cloud.

---

## 2. API Contract

### Interfaz Principal

```python
class MetadataEngine(ABC):
    def extract_text(self, path: Path) -> ExtractedText: pass
    def summarize(self, path: Path, summary_type: SummaryType) -> SummaryResult: pass
    def suggest_labels(self, path: Path) -> LabelSuggestion: pass
    def get_preview(self, path: Path) -> PreviewResult: pass
    def generate_sidecar(self, path: Path) -> SidecarFile: pass
```

### DTOs Principales

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
    summary_type: SummaryType  # SHORT o LONG
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

> Schema JSON detallado: ver [SPEC_SIDECAR.md](SPEC_SIDECAR.md)

```python
# DTOs internos para SidecarFile
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

## 3. Arquitectura

### Patrón Adapter

```
fileKor Core
       ↓
   Adapter (interfaz común)
       ↓
   External Tool (pdfplumber, PyExifTool, embeddings, LLM)
```

### Stack Tecnológico

| Paquete | Rol | License |
|---------|-----|---------|
| pdfplumber | Extracción texto PDFs | MIT |
| PyExifTool | Extracción metadatos | GPL/BSD |
| click | Framework CLI | BSD-3-Clause |
| rich | Output formateado | MIT |
| pydantic | Validación DTOs | MIT |

---

## 4. Modelo de Capas

Ver documento completo: [SPEC_LAYERS.md](SPEC_LAYERS.md)

| Layer | Nombre | Descripción | Costo |
|-------|--------|-------------|-------|
| **L0** | Hash Dedup | SHA256 cache check | $0 |
| **L1** | Native + OS | Metadata nativa (exiftool) | $0 |
| **L2** | Embeddings | Similarity check (adapter pattern) | $0 |
| **L3** | LLM Prompt | Gemini + Ollama (adapter pattern) | $ |

### Niveles de Análisis

| Nivel | Capas | Costo | Precisión |
|-------|-------|-------|-----------|
| **minimal** | L0 | $0 | Baja |
| **fast** | L0 + L1 | $0 | Media |
| **standard** | L0 + L1 + L2 | $0 | Alta |
| **deep** | Todas | $ | Muy Alta |

### Confidence Threshold

- Hardcodeado en Phase 0 (ej: 0.8)
- Configurable via `config.yaml` en fases posteriores

---

## 5. Sidecar Schema

Ver documento completo: [SPEC_SIDECAR.md](SPEC_SIDECAR.md)

---

## 6. Taxonomía de Labels

Ver documento completo: [SPEC_TAXONOMY.md](SPEC_TAXONOMY.md)

- **Base predefinida:** `contract, finance, legal, provider, architecture, config, notes`
- **Configurable:** Via archivo del consumidor

---

## 7. Manejo de Errores

| Parser Status | Significado |
|---------------|-------------|
| `OK` | Extraído correctamente |
| `DEGRADED` | Encriptado o parcial — `"Needs OCR or password"` |
| `BROKEN` | No se puede leer |

**Filosofía:** No intentar descifrar, marcar y continuar.

---

## 8. Construcción por Fases

Ver documento completo: [SPEC_PHASES.md](SPEC_PHASES.md)

### Phase 0 — Fundamentos (Testeable)
- Layer 0: Hash + cache (stdlib)
- Layer 1: Adapter + Mock
- Tests unitarios

### Phase 1 — Metadata Real
- PyExifTool real
- Sidecar generation
- Caché incremental

### Phase 2 — Embeddings 
- EmbeddingsAdapter interface
- Mock + real implementation

### Phase 3 — LLM 
- LLMAdapter interface
- Gemini + Ollama implementations

---

## 9. Principios de Desarrollo

- **Mock-first:** Construir contra interfaces
- **Adapter pattern:** No acoplar a herramientas específicas
- **Testeable:** Cada layer probable independientemente
- **Local-only:** Sin cloud services
- **Construcción por capas:** No agregar complejidad antes de tiempo

---

## 10. Referencias

| Documento | Propósito |
|-----------|-----------|
| [SPEC_LAYERS.md](SPEC_LAYERS.md) | Modelo de capas detallado |
| [SPEC_SIDECAR.md](SPEC_SIDECAR.md) | Sidecar schema completo |
| [SPEC_TAXONOMY.md](SPEC_TAXONOMY.md) | Taxonomía de labels |
| [SPEC_PHASES.md](SPEC_PHASES.md) | Plan de construcción por fases |
| [SPEC_CLI.md](SPEC_CLI.md) | Interface CLI detallada |
| questions.md | Decisiones de diseño respondidas |

---

## 11. Decisiones Tomadas

| # | Decisión |
|---|----------|
| D001 | pdfplumber para extracción de texto |
| D002 | PyExifTool para metadatos |
| D003 | Taxonomía híbrida (predefinida + configurable) |
| D004 | Sidecars + caché para persistencia |
| D005 | Extracción incremental con hash comparison |
| D006 | Archivos encriptados → DEGRADED |
| D007 | Modelo por capas para summaries |
| D008 | CLI + PyPI como modos de consumo |
| D009 | Layer 2 usa Adapter pattern (cualquier librería) |
| D010 | Layer 3 usa Gemini + Ollama (Adapter pattern) |
| D011 | Confidence threshold hardcodeado + configurable |
| D012 | Sidecar schema deduce del FileRecord del consumidor |
| D013 | Archivos de metadata se llaman `.kor` (ej: `archivo.kor`) |
