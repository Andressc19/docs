# SPEC: Layer Model

**Document:** Layered processing model to reduce AI costs  
**Reference:** INIT_SPEC.md section 4

---

## Layer Architecture

```
┌─────────────────────────────────────────────────┐
│  fileKor Core                                   │
├─────────────────────────────────────────────────┤
│  Layer 0: Hash Dedup ──────────────────────────►│ If cache hit → return
│  Layer 1: Native + OS ────────────────────────► │ If confidence > 0.8 → return
│  Layer 2: Embeddings (Adapter) ───────────────► │ If similarity > 0.9 → return
│  Layer 3: LLM (Adapter) ← Gemini, Ollama      │
└─────────────────────────────────────────────────┘
```

---

## Details by Layer

### Layer 0 — Hash Dedup

| Aspect | Description |
|--------|-------------|
| **Function** | Calculate SHA256 and check cache |
| **Dependencies** | stdlib (hashlib) |
| **Cost** | $0 |
| **Latency** | <10ms |
| **Testable** | ✅ Easy (deterministic) |

```python
# If file exists in cache → return cached result
# If not → continue to Layer 1
```

---

### Layer 1 — Native + OS Metadata

| Aspect | Description |
|--------|-------------|
| **Function** | Extract native file metadata |
| **Dependencies** | PyExifTool |
| **Cost** | $0 |
| **Latency** | <100ms |
| **Testable** | ✅ Mockable |

**Extracts:**
- filename, extension, size
- timestamps (created, modified)
- author, title (from PDF/PDF props)
- parent folder, path keywords

**Default confidence:** 0.8

```python
# If confidence >= threshold → return Layer 1 result
# If not → continue to Layer 2
```

---

### Layer 2 — Embeddings (Adapter Pattern)

| Aspect | Description |
|--------|-------------|
| **Function** | Find similar files via embeddings |
| **Dependencies** | Adapter (sentence-transformers, fastembed, mock) |
| **Cost** | $0 |
| **Latency** | <500ms |
| **Testable** | ⚠️ Requires mocks |

**Adapter Interface:**

```python
class EmbeddingsAdapter(ABC):
    @abstractmethod
    def get_embedding(self, text: str) -> List[float]: pass
    
    @abstractmethod
    def find_similar(self, embedding: List[float], threshold: float) -> List[SimilarFile]: pass
```

**Implementations:**

| Implementation | Library | Status |
|---------------|---------|--------|
| MockEmbeddingsImpl | N/A | ✅ Phase 0 |
| SentenceTransformersImpl | sentence-transformers | Phase 2 |
| FastEmbedImpl | fastembed | Phase 2 |

```python
# If similarity >= threshold → return similar file metadata
# If not → continue to Layer 3
```

---

### Layer 3 — LLM (Adapter Pattern)

| Aspect | Description |
|--------|-------------|
| **Function** | Complete semantic analysis via LLM |
| **Dependencies** | Adapter (Gemini, Ollama, mock) |
| **Cost** | Gemini: ~$0 (free tier), Ollama: $0 |
| **Latency** | 1-3s per file |
| **Testable** | ⚠️ Mandatory mock |

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

**Implementations:**

| Implementation | Provider | Status |
|---------------|----------|--------|
| MockLLMImpl | N/A | ✅ Phase 0 |
| GeminiImpl | Google Gemini | Phase 3 |
| OllamaImpl | Ollama (local) | Phase 3 |

---

## Analysis Levels

| Level | Activated Layers | Cost | Precision | Latency |
|-------|------------------|------|-----------|---------|
| **minimal** | L0 | $0 | Low | <10ms |
| **fast** | L0 + L1 | $0 | Medium | <100ms |
| **standard** | L0 + L1 + L2 | $0 | High | <500ms |
| **deep** | L0 + L1 + L2 + L3 | $ | Very High | 1-3s |

---

## Confidence Thresholds

```python
class Config:
    # Hardcoded in Phase 0
    LAYER_1_CONFIDENCE_THRESHOLD = 0.8
    LAYER_2_SIMILARITY_THRESHOLD = 0.9
    
    # Configurable in later phases
    layer_1_confidence_threshold: float = 0.8
    layer_2_similarity_threshold: float = 0.9
```

---

## Pipeline Implementation

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

## Layer Tests

```python
# tests/test_layers/
# ├── test_layer_0_hash.py
# ├── test_layer_1_metadata.py
# ├── test_layer_2_embeddings.py
# └── test_layer_3_llm.py

def test_layer_0_returns_cached():
    """Layer 0 returns cached if exists"""
    pass

def test_layer_1_shortcut_on_high_confidence():
    """Layer 1 shortcuts if confidence > threshold"""
    pass

def test_layer_2_shortcut_on_high_similarity():
    """Layer 2 uses cache if similarity > threshold"""
    pass

def test_layer_3_called_only_when_needed():
    """Layer 3 only called if previous layers didn't satisfy"""
    pass
```