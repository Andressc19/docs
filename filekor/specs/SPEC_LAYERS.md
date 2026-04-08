# SPEC: Modelo de Capas

**Documento:** Modelo de procesamiento por capas para reducir costos de IA  
**Referencia:** INIT_SPEC.md sección 4

---

## Arquitectura de Capas

```
┌─────────────────────────────────────────────────┐
│  fileKor Core                                   │
├─────────────────────────────────────────────────┤
│  Layer 0: Hash Dedup ──────────────────────────►│ Si cache hit → return
│  Layer 1: Native + OS ────────────────────────► │ Si confidence > 0.8 → return
│  Layer 2: Embeddings (Adapter) ───────────────► │ Si similarity > 0.9 → return
│  Layer 3: LLM (Adapter) ← Gemini, Ollama      │
└─────────────────────────────────────────────────┘
```

---

## Detalle por Layer

### Layer 0 — Hash Dedup

| Aspecto | Descripción |
|---------|-------------|
| **Función** | Calcular SHA256 y verificar caché |
| **Dependencias** | stdlib (hashlib) |
| **Costo** | $0 |
| **Latencia** | <10ms |
| **Testeable** | ✅ Fácil (determinista) |

```python
# Si archivo existe en cache → return cached result
# Si no → continue to Layer 1
```

---

### Layer 1 — Native + OS Metadata

| Aspecto | Descripción |
|---------|-------------|
| **Función** | Extraer metadata nativa del archivo |
| **Dependencias** | PyExifTool |
| **Costo** | $0 |
| **Latencia** | <100ms |
| **Testeable** | ✅ Mockeable |

**Extrae:**
- filename, extension, size
- timestamps (created, modified)
- author, title (desde PDF/PDF props)
- parent folder, path keywords

**Confidence default:** 0.8

```python
# Si confidence >= threshold → return Layer 1 result
# Si no → continue to Layer 2
```

---

### Layer 2 — Embeddings (Adapter Pattern)

| Aspecto | Descripción |
|---------|-------------|
| **Función** | Buscar archivos similares via embeddings |
| **Dependencias** | Adapter (sentence-transformers, fastembed, mock) |
| **Costo** | $0 |
| **Latencia** | <500ms |
| **Testeable** | ⚠️ Requiere mocks |

**Adapter Interface:**

```python
class EmbeddingsAdapter(ABC):
    @abstractmethod
    def get_embedding(self, text: str) -> List[float]: pass
    
    @abstractmethod
    def find_similar(self, embedding: List[float], threshold: float) -> List[SimilarFile]: pass
```

**Implementaciones:**

| Implementation | Librería | Status |
|---------------|----------|--------|
| MockEmbeddingsImpl | N/A | ✅ Phase 0 |
| SentenceTransformersImpl | sentence-transformers | Phase 2 |
| FastEmbedImpl | fastembed | Phase 2 |

```python
# Si similarity >= threshold → return similar file metadata
# Si no → continue to Layer 3
```

---

### Layer 3 — LLM (Adapter Pattern)

| Aspecto | Descripción |
|---------|-------------|
| **Función** | Análisis semántico completo via LLM |
| **Dependencias** | Adapter (Gemini, Ollama, mock) |
| **Costo** | Gemini: ~$0 (free tier), Ollama: $0 |
| **Latencia** | 1-3s por archivo |
| **Testeable** | ⚠️ Mock obligatorio |

**Adapter Interface:**

```python
class LLMAdapter(ABC):
    @abstractmethod
    def analyze(self, text: str, context: dict) -> LLMAnalysis: pass
    
    @abstractmethod
    def summarize(self, text: str, summary_type: SummaryType) -> Summary: pass
    
    @abstractmethod
    def suggest_labels(self, text: str, taxonomy: List[str]) -> LabelSuggestion: pass
```

**Implementaciones:**

| Implementation | Proveedor | Status |
|---------------|-----------|--------|
| MockLLMImpl | N/A | ✅ Phase 0 |
| GeminiImpl | Google Gemini | Phase 3 |
| OllamaImpl | Ollama (local) | Phase 3 |

---

## Niveles de Análisis

| Nivel | Capas Activadas | Costo | Precisión | Latencia |
|-------|-----------------|-------|-----------|----------|
| **minimal** | L0 | $0 | Baja | <10ms |
| **fast** | L0 + L1 | $0 | Media | <100ms |
| **standard** | L0 + L1 + L2 | $0 | Alta | <500ms |
| **deep** | L0 + L1 + L2 + L3 | $ | Muy Alta | 1-3s |

---

## Confidence Thresholds

```python
class Config:
    # Hardcodeado en Phase 0
    LAYER_1_CONFIDENCE_THRESHOLD = 0.8
    LAYER_2_SIMILARITY_THRESHOLD = 0.9
    
    # Configurable en fases posteriores
    layer_1_confidence_threshold: float = 0.8
    layer_2_similarity_threshold: float = 0.9
```

---

## Implementación del Pipeline

```python
class MetadataProcessor:
    def __init__(self, config: ProcessorConfig):
        self.config = config
        self.layer_0 = Layer0Hash()
        self.layer_1 = Layer1Metadata(config.metadata_adapter)
        self.layer_2 = Layer2Embeddings(config.embeddings_adapter)
        self.layer_3 = Layer3LLM(config.llm_adapter)
    
    def process(self, file_path: Path) -> ProcessingResult:
        result = ProcessingResult(file_path=file_path)
        
        # Layer 0: Hash check
        if cached := self.layer_0.check_cache(file_path):
            return cached
        
        # Layer 1: Native metadata
        if self.config.use_layer_1:
            metadata = self.layer_1.extract(file_path)
            if metadata.confidence >= self.config.layer_1_threshold:
                result.layers_used = ["0", "1"]
                result.metadata = metadata
                return result
        
        # Layer 2: Embeddings
        if self.config.use_layer_2:
            similar = self.layer_2.find_similar(metadata)
            if similar and similar.confidence >= self.config.layer_2_threshold:
                result.layers_used = ["0", "1", "2"]
                result.metadata = similar
                return result
        
        # Layer 3: LLM
        if self.config.use_layer_3:
            result.layers_used = ["0", "1", "2", "3"]
            result.metadata = self.layer_3.analyze(metadata)
        
        return result
```

---

## Tests por Layer

```python
# tests/test_layers/
# ├── test_layer_0_hash.py
# ├── test_layer_1_metadata.py
# ├── test_layer_2_embeddings.py
# └── test_layer_3_llm.py

def test_layer_0_returns_cached():
    """Layer 0 retorna cached si existe"""
    pass

def test_layer_1_shortcut_on_high_confidence():
    """Layer 1 hace shortcut si confidence > threshold"""
    pass

def test_layer_2_shortcut_on_high_similarity():
    """Layer 2 usa cache si similarity > threshold"""
    pass

def test_layer_3_called_only_when_needed():
    """Layer 3 solo se llama si capas anteriores no satisficieron"""
    pass
```
