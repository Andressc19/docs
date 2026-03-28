# Issue 7: Niveles de Análisis Configurables

**Estado:** Pendiente  
**Prioridad:** Baja  
**Estimación:** 1-2 días

---

## Descripción

Permitir configurar el nivel de análisis según el caso de uso: desde análisis rápido (costo $0) hasta análisis profundo (máxima precisión).

---

## Niveles de Análisis

| Nivel | Capas Activadas | Costo | Precisión | Latencia |
|-------|-----------------|-------|-----------|----------|
| **minimal** | Layer 0 (hash) | $0 | Baja | <10ms |
| **fast** | Layer 0 + 0.5 (native) | $0 | Media | <50ms |
| **standard** | Layer 0 + 0.5 + 2 + 3 | $0 | Alta | <200ms |
| **deep** | Todas (0-4) | $ | Muy Alta | 1-3s/arch |

---

## Detalle por Nivel

### minimal

```
✓ Layer 0: Hash dedup
✗ Layer 0.5: Native metadata
✗ Layer 1: OS context (solo filename)
✗ Layer 2: Clustering
✗ Layer 3: Embeddings
✗ Layer 4: LLM
```

**Caso de uso:** Pre-indexación rápida, detección de duplicados exactos.

---

### fast

```
✓ Layer 0: Hash dedup
✓ Layer 0.5: Native metadata
✗ Layer 1: OS context (completo)
✗ Layer 2: Clustering
✗ Layer 3: Embeddings
✗ Layer 4: LLM
```

**Caso de uso:**快速escaneo, metadata básica.

---

### standard (default)

```
✓ Layer 0: Hash dedup
✓ Layer 0.5: Native metadata
✓ Layer 1: OS context
✓ Layer 2: Clustering
✓ Layer 3: Embeddings
✗ Layer 4: LLM
```

**Caso de uso:** Balance costo/precisión, cache hit.

---

### deep

```
✓ Layer 0: Hash dedup
✓ Layer 0.5: Native metadata
✓ Layer 1: OS context
✓ Layer 2: Clustering
✓ Layer 3: Embeddings
✓ Layer 4: LLM
```

**Caso de uso:** Máxima precisión, análisis de contenido profundo.

---

## Implementación

```python
# sdk/config.py
from enum import Enum

class AnalysisLevel(Enum):
    MINIMAL = "minimal"
    FAST = "fast"
    STANDARD = "standard"
    DEEP = "deep"

@dataclass
class ProcessorConfig:
    level: AnalysisLevel = AnalysisLevel.STANDARD
    
    # Layer flags
    use_layer_0: bool = True
    use_layer_0_5: bool = True
    use_layer_1: bool = True
    use_layer_2: bool = True
    use_layer_3: bool = True
    use_layer_4: bool = True
    
    @classmethod
    def from_level(cls, level: AnalysisLevel) -> "ProcessorConfig":
        config = cls(level=level)
        
        if level == AnalysisLevel.MINIMAL:
            config.use_layer_0_5 = False
            config.use_layer_1 = False
            config.use_layer_2 = False
            config.use_layer_3 = False
            config.use_layer_4 = False
        
        elif level == AnalysisLevel.FAST:
            config.use_layer_0_5 = True
            config.use_layer_1 = False
            config.use_layer_2 = False
            config.use_layer_3 = False
            config.use_layer_4 = False
        
        elif level == AnalysisLevel.STANDARD:
            config.use_layer_0_5 = True
            config.use_layer_1 = True
            config.use_layer_2 = True
            config.use_layer_3 = True
            config.use_layer_4 = False
        
        elif level == AnalysisLevel.DEEP:
            pass  # All True
        
        return config
```

---

```python
# sdk/processor.py
class MetadataProcessor:
    def __init__(self, config: ProcessorConfig = None):
        self.config = config or ProcessorConfig()
    
    def extract_metadata(self, file_path: str) -> dict:
        # Layer 0
        if self.config.use_layer_0:
            result = self._layer_0_hash(file_path)
            if result:
                return result
        
        # Layer 0.5
        if self.config.use_layer_0_5:
            result = self._layer_0_5_native(file_path)
            if result:
                return result
        
        # Layer 1
        if self.config.use_layer_1:
            local_context = self._layer_1_context(file_path)
        
        # ... y así sucesivamente
        
        # Layer 4
        if self.config.use_layer_4:
            result = self._layer_4_llm(...)
            return result
        
        # Si Layer 4 desactivado pero se necesita
        return self._fallback_metadata(file_path)
```

---

## CLI

```bash
# Análisis rápido (minimal cost)
sidecar-tagger scan /Docs --level fast

# Análisis profundo (máxima precisión)
sidecar-tagger scan /Docs --level deep

# Default (standard)
sidecar-tagger scan /Docs  # level=standard por defecto

# Ver niveles disponibles
sidecar-tagger --help | grep -A5 "level"
```

---

## Casos de Uso

### 1. Pre-indexación periódica (fast)

```bash
# Cron job semanal - escaneo rápido
0 2 * * 0 sidecar-tagger scan /Documents --level fast --quiet
```

### 2. Indexación inicial (standard)

```bash
# Primera vez - análisis completo sin LLM
sidecar-tagger scan /Documents --level standard
```

### 3. Análisis de archivos importantes (deep)

```bash
# Archivos seleccionados - análisis profundo
sidecar-tagger analyze --level deep file1.pdf file2.pdf
```

---

## Configuración Persistente

```yaml
# ~/.sidecar-tagger/config.yaml
default_level: standard

# Override por path
path_overrides:
  /Documents/Archive: minimal
  /Documents/Important: deep
```

---

## Tests Recomendados

- `test_level_minimal()` - Solo hash
- `test_level_fast()` - Hasta native metadata
- `test_level_standard()` - Hasta embeddings
- `test_level_deep()` - Full pipeline
- `test_config_from_level()` - Factory correcto

---

## Métricas por Nivel

| Nivel | Costo/1000 archivos | Tiempo |
|-------|-------------------|--------|
| minimal | $0 | ~5s |
| fast | $0 | ~30s |
| standard | $0 | ~2min |
| deep | ~$10 | ~30min |
