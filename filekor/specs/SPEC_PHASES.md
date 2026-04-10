# SPEC: Development Phases

**Document:** Phased construction plan for incremental testing  
**Reference:** INIT_SPEC.md section 8

---

## General Principle

> Build layer by layer, don't add complexity before its time. Each phase must be testable independently.

---

## Phase 0 — Fundamentals (Testable)

**Objective:** Create base structure + Layer 0 + Layer 1 mock

### Estimated duration: 1-2 days

### Deliverables:

- [ ] Project structure (poetry/pyproject.toml)
- [ ] Layer 0: Hash + cache check (stdlib only)
- [ ] Layer 1 Adapter interface
- [ ] MockMetadataAdapter (realistic data)
- [ ] Unit tests for L0 and L1
- [ ] Basic functional CLI
- [ ] Sidecar schema v1

### Cost: $0

### Exit criterion: Tests pass without external dependencies

---

## Phase 1 — Real Metadata

**Objective:** Integrate real PyExifTool

### Estimated duration: 2-3 days

### Deliverables:

- [ ] Real PyExifTool integration
- [ ] Integration tests L0 + L1
- [ ] Functional sidecar generation
- [ ] Incremental cache (hash comparison)
- [ ] Complete CLI for extract/preview/labels/sidecar

### Cost: $0

### Exit criterion: I can process real PDFs and generate sidecars

---

## Phase 2 — Embeddings (Optional)

**Objective:** Add similarity search

### Estimated duration: 1-2 days

### Deliverables:

- [ ] EmbeddingsAdapter interface
- [ ] MockEmbeddingsImpl (tests)
- [ ] SentenceTransformersImpl or FastEmbedImpl
- [ ] Similarity check tests
- [ ] Cache integration

### Cost: $0

### Exit criterion: I can find similar files by content

---

## Phase 3 — LLM (Plus)

**Objective:** Complete semantic analysis

### Estimated duration: 2-3 days

### Deliverables:

- [ ] LLMAdapter interface
- [ ] MockLLMImpl (tests)
- [ ] GeminiImpl
- [ ] OllamaImpl
- [ ] Configurable confidence threshold
- [ ] Prompt tests

### Cost: ~$0 (Gemini free tier) or $0 (Ollama local)

### Exit criterion: I can generate long summaries and labels with high confidence

---

## Phases Summary

| Phase | Layers | Duration | Cost | Value |
|-------|--------|----------|------|-------|
| **Phase 0** | L0 + L1 mock | 1-2 days | $0 | Functional MVP without AI |
| **Phase 1** | L1 real | 2-3 days | $0 | Real metadata extraction |
| **Phase 2** | L2 | 1-2 days | $0 | Similarity search |
| **Phase 3** | L3 | 2-3 days | ~$0 | LLM + deep analysis |

**Total:** ~6-10 days for complete product

---

## Test Structure

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

## Exit Criteria Checklist

### Phase 0

```python
def test_phase_0_exit():
    # No API keys, no external dependencies
    processor = MetadataProcessor(MockAdapter())
    
    # Process file and generate valid sidecar
    result = processor.process("tests/fixtures/sample.pdf")
    
    assert result.layers_used == ["0", "1"]
    assert result.metadata.confidence >= 0.8
    assert SidecarSchema.validate(result.sidecar)
```

### Phase 1

```python
def test_phase_1_exit():
    # With real PyExifTool
    processor = MetadataProcessor(PyExifToolAdapter())
    
    # Process real PDF
    result = processor.process("real_document.pdf")
    
    assert result.metadata.author is not None
    assert result.sidecar["metadata"]["pages"] > 0
```

---

## Rollback Plan

If a phase fails to complete:

| Phase | Fallback |
|-------|----------|
| Phase 0 | Reposition as "spec + mocks" |
| Phase 1 | Keep with mocks |
| Phase 2 | Skip, use only L0+L1 |
| Phase 3 | Skip, use L1 confidence |

---

## Notes

- Each phase should be merged to main independently
- Use feature flags to enable/disable layers
- Document decisions from each phase