# SPEC: Fases de Desarrollo

**Documento:** Plan de construcción por fases para testing incremental  
**Referencia:** INIT_SPEC.md sección 8

---

## Principio General

> Construir capa por capa, no agregar complejidad antes de tiempo. Cada fase debe ser testeable independientemente.

---

## Phase 0 — Fundamentos (Testeable)

**Objetivo:** Crear estructura base + Layer 0 + Layer 1 mock

### Duración estimada: 1-2 días

### Entregables:

- [ ] Estructura del proyecto (poetry/pyproject.toml)
- [ ] Layer 0: Hash + cache check (stdlib only)
- [ ] Layer 1 Adapter interface
- [ ] MockMetadataAdapter (datos realistas)
- [ ] Tests unitarios para L0 y L1
- [ ] CLI básica funcional
- [ ] Sidecar schema v1

### Costo: $0

### Criterio de exit: Tests pasan sin dependencias externas

---

## Phase 1 — Metadata Real

**Objetivo:** Integrar PyExifTool real

### Duración estimada: 2-3 días

### Entregables:

- [ ] PyExifTool real integration
- [ ] Tests de integración L0 + L1
- [ ] Sidecar generation funcional
- [ ] Caché incremental (hash comparison)
- [ ] CLI completa para extract/preview/labels/sidecar

### Costo: $0

### Criterio de exit: Puedo procesar PDFs reales y generar sidecars

---

## Phase 2 — Embeddings (Opcional)

**Objetivo:** Agregar similarity search

### Duración estimada: 1-2 días

### Entregables:

- [ ] EmbeddingsAdapter interface
- [ ] MockEmbeddingsImpl (tests)
- [ ] SentenceTransformersImpl o FastEmbedImpl
- [ ] Tests de similarity check
- [ ] Integración con cache

### Costo: $0

### Criterio de exit: Puedo encontrar archivos similares por contenido

---

## Phase 3 — LLM (Plus)

**Objetivo:** Análisis semántico completo

### Duración estimada: 2-3 días

### Entregables:

- [ ] LLMAdapter interface
- [ ] MockLLMImpl (tests)
- [ ] GeminiImpl
- [ ] OllamaImpl
- [ ] Confidence threshold configurable
- [ ] Tests de prompts

### Costo: ~$0 (Gemini free tier) o $0 (Ollama local)

### Criterio de exit: Puedo generar summaries largos y labels con alta confianza

---

## Resumen de Fases

| Fase | Capas | Duración | Costo | Valor |
|------|-------|----------|-------|-------|
| **Phase 0** | L0 + L1 mock | 1-2 días | $0 | MVP funcional sin IA |
| **Phase 1** | L1 real | 2-3 días | $0 | Metadata extraction real |
| **Phase 2** | L2 | 1-2 días | $0 | Similarity search |
| **Phase 3** | L3 | 2-3 días | ~$0 | LLM + deep analysis |

**Total:** ~6-10 días para producto completo

---

## Estructura de Tests

```
tests/
├── unit/
│   ├── test_layer_0_hash.py
│   ├── test_layer_1_metadata.py
│   ├── test_sidecar_schema.py
│   └── test_adapters.py
├── integration/
│   ├── test_layer_0_1_pipeline.py
│   └── test_cache.py
└── fixtures/
    ├── sample.pdf
    └── mock_metadata.json
```

---

## Checklist de Criterio de Exit

### Phase 0

```python
def test_phase_0_exit():
    # Sin API keys, sin dependencias externas
    processor = MetadataProcessor(MockAdapter())
    
    # Procesa archivo y genera sidecar válido
    result = processor.process("tests/fixtures/sample.pdf")
    
    assert result.layers_used == ["0", "1"]
    assert result.metadata.confidence >= 0.8
    assert SidecarSchema.validate(result.sidecar)
```

### Phase 1

```python
def test_phase_1_exit():
    # Con PyExifTool real
    processor = MetadataProcessor(PyExifToolAdapter())
    
    # Procesa PDF real
    result = processor.process("real_document.pdf")
    
    assert result.metadata.author is not None
    assert result.sidecar["metadata"]["pages"] > 0
```

---

## Rollback Plan

Si una fase falla en completar:

| Fase | Fallback |
|------|----------|
| Phase 0 | Reposicionar como "spec + mocks" |
| Phase 1 | Mantener con mocks |
| Phase 2 | Omitir, usar solo L0+L1 |
| Phase 3 | Omitir, usar confidence de L1 |

---

## Notas

- Cada fase debe mergearse a main independently
- Usar feature flags para activar/desactivar capas
- Documentar decisiones de cada fase
