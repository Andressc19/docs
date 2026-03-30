# Issue 1: Niveles de Análisis y Tests de Integración

**Estado:** Pendiente  
**Prioridad:** Alta  
**Estimación:** 3-4 días

---

## Descripción

Implementar niveles de análisis configurables y tests de integración para el pipeline de 4 capas.

---

## Estructura de Capas

| Layer | Nombre | Descripción |
|-------|--------|-------------|
| **Layer 0** | Hash Dedup | SHA256 - si existe, return cached |
| **Layer 1** | Native + OS | EXIFTOOL + filename, path, size, timestamps |
| **Layer 2** | Embeddings | FastEmbed - similarity check (cache hit → return) |
| **Layer 3** | LLM Prompt | Gemini + Clustering Hint para contexto |

---

## Niveles de Análisis

| Nivel | Capas Activadas | Costo | Precisión | Latencia |
|-------|-----------------|-------|-----------|----------|
| **minimal** | Layer 0 | $0 | Baja | <10ms |
| **fast** | Layer 0 + 1 | $0 | Media | <100ms |
| **standard** | Layer 0 + 1 + 2 | $0 | Alta | <500ms |
| **deep** | Todas (0-3) | $ | Muy Alta | 1-3s/arch |

---

## Detalle por Nivel

### minimal

```
✓ Layer 0: Hash dedup
✗ Layer 1: Native + OS
✗ Layer 2: Embeddings
✗ Layer 3: LLM
```

**Caso de uso:** Pre-indexación rápida, solo detección de duplicados.

---

### fast

```
✓ Layer 0: Hash dedup
✓ Layer 1: Native + OS (EXIFTOOL)
✗ Layer 2: Embeddings
✗ Layer 3: LLM
```

**Caso de uso:** Escaneo rápido con metadata básica.

---

### standard (default)

```
✓ Layer 0: Hash dedup
✓ Layer 1: Native + OS
✓ Layer 2: Embeddings (cache hit)
✗ Layer 3: LLM
```

**Caso de uso:** Balance costo/precisión, reuse de metadata.

---

### deep

```
✓ Layer 0: Hash dedup
✓ Layer 1: Native + OS
✓ Layer 2: Embeddings
✓ Layer 3: LLM + Clustering Hint
```

**Caso de uso:** Máxima precisión.

---

## Implementación

```python
# sdk/config.py
from enum import Enum
from dataclasses import dataclass

class AnalysisLevel(Enum):
    MINIMAL = "minimal"
    FAST = "fast"
    STANDARD = "standard"
    DEEP = "deep"

@dataclass
class ProcessorConfig:
    level: AnalysisLevel = AnalysisLevel.STANDARD
    
    use_layer_0: bool = True
    use_layer_1: bool = True
    use_layer_2: bool = True
    use_layer_3: bool = True
    
    @classmethod
    def from_level(cls, level: AnalysisLevel) -> "ProcessorConfig":
        config = cls(level=level)
        
        if level == AnalysisLevel.MINIMAL:
            config.use_layer_1 = False
            config.use_layer_2 = False
            config.use_layer_3 = False
        
        elif level == AnalysisLevel.FAST:
            config.use_layer_2 = False
            config.use_layer_3 = False
        
        elif level == AnalysisLevel.STANDARD:
            config.use_layer_3 = False
        
        return config
```

---

## Processor

```python
# sdk/processor.py
class MetadataProcessor:
    def __init__(self, config: ProcessorConfig = None):
        self.config = config or ProcessorConfig()
        self.layer_0 = Layer0()
        self.layer_1 = Layer1()      # Native + OS
        self.layer_2 = Layer2()       # Embeddings
        self.layer_3 = Layer3()       # LLM + Clustering
    
    def process(self, file_path: str) -> dict:
        result = {"file": file_path, "layers": []}
        
        # Layer 0: Hash
        if self.config.use_layer_0:
            cached = self.layer_0.process(file_path)
            if cached:
                result["metadata"] = cached
                result["layers"].append("0")
                return result
        
        # Layer 1: Native + OS
        if self.config.use_layer_1:
            native = self.layer_1.process(file_path)
            if native and native.confidence >= 0.8:
                result["metadata"] = native
                result["layers"].append("1")
                return result
        
        # Layer 2: Embeddings
        if self.config.use_layer_2:
            similar = self.layer_2.process(file_path)
            if similar:
                result["metadata"] = similar
                result["layers"].append("2")
                return result
        
        # Layer 3: LLM + Clustering Hint
        if self.config.use_layer_3:
            clustering_hint = self.layer_3.get_clustering_hint(file_path)
            llm_result = self.layer_3.process(file_path, clustering_hint)
            result["metadata"] = llm_result
            result["layers"].append("3")
        
        return result
```

---

## CLI

```bash
# Análisis rápido
sidecar-tagger scan /Docs --level fast

# Análisis profundo
sidecar-tagger scan /Docs --level deep

# Default (standard)
sidecar-tagger scan /Docs
```

---

## Tests de Integración

```python
# tests/test_pipeline.py

def test_layer_0_hash_dedup():
    """Layer 0 detecta duplicados"""
    # ...

def test_layer_1_native_os():
    """Layer 1 extrae native + OS metadata"""
    # ...

def test_layer_2_embeddings():
    """Layer 2 usa embeddings para cache"""
    # ...

def test_layer_3_llm_with_hint():
    """Layer 3 usa clustering hint en prompt"""
    # ...


def test_level_minimal():
    config = ProcessorConfig.from_level(AnalysisLevel.MINIMAL)
    processor = MetadataProcessor(config)
    result = processor.process("test.pdf")
    assert "0" in result["layers"]
    assert "1" not in result["layers"]

def test_level_fast():
    config = ProcessorConfig.from_level(AnalysisLevel.FAST)
    processor = MetadataProcessor(config)
    result = processor.process("test.pdf")
    assert "0" in result["layers"]
    assert "1" in result["layers"]

def test_level_standard():
    config = ProcessorConfig.from_level(AnalysisLevel.STANDARD)
    processor = MetadataProcessor(config)
    result = processor.process("test.pdf")
    assert "0" in result["layers"]
    assert "1" in result["layers"]
    assert "2" in result["layers"]

def test_level_deep():
    config = ProcessorConfig.from_level(AnalysisLevel.DEEP)
    processor = MetadataProcessor(config)
    result = processor.process("test.pdf")
    assert result["layers"] == ["0", "1", "2", "3"]


def test_shortcut_layer_1():
    """Si confidence > 0.8, hace shortcut"""
    # ...

def test_shortcut_layer_2():
    """Si similarity > 0.9, usa cache"""
    # ...
```

---

## Métricas por Nivel

| Nivel | Costo/1000 archivos | Tiempo |
|-------|-------------------|--------|
| minimal | $0 | ~5s |
| fast | $0 | ~30s |
| standard | $0 | ~2min |
| deep | ~$10 | ~30min |

---

## Referencias

- [MVP.md](../mvp/MVP.md)
- [file-extensions.md](../mvp/file-extensions.md)
