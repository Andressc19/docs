# Tasks: fileKor Implementation

## Phase 1: Project Foundation

- [x] 1.1 Create `pyproject.toml` with project metadata and dependencies
- [x] 1.2 Create `filekor/__init__.py` with package exports and get_capabilities()
- [x] 1.3 Create `filekor/core.py` with all dataclasses (ExtractedText, SummaryResult, etc.)
- [x] 1.4 Create `filekor/core.py` with MetadataEngine abstract class
- [x] 1.5 Create `filekor/core.py` with ProcessorConfig dataclass
- [x] 1.6 Create `filekor/core.py` with Layer0Hash, Layer1Metadata, Layer2Embeddings, Layer3LLM stub classes
- [x] 1.7 Create `filekor/adapters/__init__.py` package init
- [x] 1.8 Create `filekor/adapters/base.py` with Adapter abstract base class

## Phase 1b: Text Extraction

- [ ] 1.9 Create `filekor/adapters/text.py` with TextExtractorAdapter
- [ ] 1.10 Implement `extract_text()` using pdfplumber
- [ ] 1.11 Add pdfplumber as dependency in pyproject.toml

## Phase 2: Mock Adapters (Phase 0)

- [x] 2.1 Create `filekor/adapters/mock.py` with MockMetadataAdapter
- [x] 2.2 Add semi-realistic metadata generation based on filename patterns to MockMetadataAdapter
- [x] 2.3 Create `filekor/adapters/mock.py` with MockEmbeddingsAdapter
- [x] 2.4 Implement get_embedding() returning deterministic vectors in MockEmbeddingsAdapter
- [x] 2.5 Implement find_similar() returning mock similar files in MockEmbeddingsAdapter
- [x] 2.6 Create `filekor/adapters/mock.py` with MockLLMAdapter
- [x] 2.7 Implement analyze() with filename-based label inference in MockLLMAdapter
- [x] 2.8 Implement summarize() with configurable length in MockLLMAdapter
- [x] 2.9 Implement suggest_labels() with keyword matching in MockLLMAdapter

## Phase 3: Real Metadata Adapter (Phase 1)

- [x] 3.1 Create `filekor/adapters/real.py` with PyExifToolAdapter
- [x] 3.2 Implement is_available() checking PyExifTool import in PyExifToolAdapter
- [x] 3.3 Implement extract_metadata() with real exiftool calls in PyExifToolAdapter
- [x] 3.4 Add error handling for missing/malformed files in PyExifToolAdapter
- [ ] 3.5 Create `filekor/cache.py` for hash-based cache management
- [ ] 3.6 Implement incremental cache checking with SHA256 comparison

## Phase 4: Core Processor Implementation

- [ ] 4.1 Update Layer0Hash.check_cache() to use real cache system
- [ ] 4.2 Update Layer1Metadata.extract() to use adapter with confidence scoring
- [ ] 4.3 Update MetadataProcessor.process() with full layer pipeline logic
- [ ] 4.4 Implement early exit when confidence threshold met
- [ ] 4.5 Implement graceful fallback when layers unavailable

## Phase 5: Sidecar Generation

- [x] 5.1 Implement generate_sidecar() in MetadataEngine
- [x] 5.2 Create SidecarFile from all layer results
- [x] 5.3 Add JSON serialization with datetime handling
- [x] 5.4 Implement sidecar file writing to disk

## Phase 5b: Sidecar Validation

- [ ] 5.5 Implement Pydantic validation for sidecar schema
- [ ] 5.6 Add parser_status validation (OK/DEGRADED/BROKEN)

## Phase 6: CLI Implementation

- [x] 6.1 Create `filekor/cli.py` with argparse subparsers
- [x] 6.2 Implement extract command handler
- [x] 6.3 Implement summarize command handler
- [x] 6.4 Implement labels command handler
- [x] 6.5 Implement preview command handler
- [x] 6.6 Implement sidecar command handler
- [x] 6.7 Implement batch command handler with directory processing
- [ ] 6.8 Add analysis level flags (minimal/fast/standard/deep)
- [ ] 6.9 Add incremental processing flag
- [ ] 6.10 Add parallel workers support to batch command

## Phase 7: Configuration System

- [ ] 7.1 Implement config file loading from ~/.filekor.yaml
- [ ] 7.2 Add config option for confidence thresholds
- [ ] 7.3 Add config option for analysis levels
- [ ] 7.4 Add config option for cache directory
- [ ] 7.5 Add config option for custom label taxonomy

## Phase 8: Unit Tests (Test-Driven)

- [ ] 8.1 Create `tests/unit/test_core.py` with ProcessorConfig tests
- [ ] 8.2 Create `tests/unit/test_adapters.py` with Adapter interface tests
- [ ] 8.3 Write RED test for MockMetadataAdapter filename pattern matching
- [ ] 8.4 Write GREEN test making MockMetadataAdapter return pattern-based data
- [ ] 8.5 Write RED test for Layer0Hash cache miss
- [ ] 8.6 Write GREEN test for Layer0Hash cache integration
- [ ] 8.7 Write test for MetadataProcessor early exit on confidence threshold
- [ ] 8.8 Write test for graceful fallback when layer unavailable

## Phase 9: Integration Tests

- [ ] 9.1 Create `tests/integration/test_pipeline.py` for full layer pipeline
- [ ] 9.2 Create `tests/integration/test_sidecar.py` for sidecar generation
- [ ] 9.3 Create `tests/integration/test_cache.py` for cache performance
- [ ] 9.4 Add test fixtures with sample PDFs for integration testing

## Phase 10: Embeddings Layer (Phase 2)

- [ ] 10.1 Create `filekor/adapters/embeddings.py` with EmbeddingsAdapter interface
- [ ] 10.2 Add SentenceTransformersImpl to embeddings.py
- [ ] 10.3 Add FastEmbedImpl to embeddings.py
- [ ] 10.4 Update Layer2Embeddings to use real adapters
- [ ] 10.5 Add similarity threshold configuration

## Phase 11: LLM Layer (Phase 3)

- [ ] 11.1 Create `filekor/adapters/llm.py` with LLMAdapter interface
- [ ] 11.2 Add GeminiImpl to llm.py with API key handling
- [ ] 11.3 Add OllamaImpl to llm.py for local LLM
- [ ] 11.4 Update Layer3LLM to use real adapters
- [ ] 11.5 Add configurable confidence threshold for LLM results

## Phase 12: Documentation & Cleanup

- [ ] 12.1 Add docstrings to all public APIs
- [ ] 12.2 Create API usage examples in README
- [ ] 12.3 Add type hints to all function signatures
- [ ] 12.4 Verify all specs referenced correctly in documentation
- [ ] 12.5 Run full test suite and fix any failures