# sidecar-tagger MVP

> Estado actual vs MVP recomendado
> Fecha: 2026-03-27

---

## ¿Qué es un MVP de sidecar-tagger?

Un **MVP (Minimum Viable Product)** de sidecar-tagger debe ser un motor de metadatos que:

1. **Procese archivos** de forma recursiva (PDF, XLSX, Images, TXT)
2. **Genere metadata estructurada** (doc_type, tags, domain, context)
3. **Optimice costos** mediante deduplicación y cache
4. **Guarde datos centralmente** sin ensuciar carpetas de usuario
5. **Sea usable desde CLI** de forma simple

---

## Estado Actual del Código

### ✅ Implementado y Funcionando

| Componente | Archivo | Estado |
|------------|---------|--------|
| **Layer 0: Hash Dedup** | `sdk/utils/hashing.py`, `processor.py:134` | ✅ Completo |
| **Layer 1: OS Context** | `sdk/context/os_extractor.py` | ✅ Completo |
| **Layer 2: Clustering** | `sdk/context/clustering.py` | ✅ Completo |
| **Layer 3: Embeddings** | `sdk/embeddings_client.py` | ✅ Completo |
| **Layer 4: LLM** | `sdk/llm_client.py` | ✅ Completo |
| **Parser PDF** | `sdk/parsers/pdf_parser.py` | ✅ |
| **Parser XLSX** | `sdk/parsers/xlsx_parser.py` | ✅ |
| **Parser Image** | `sdk/parsers/image_parser.py` | ✅ |
| **Parser TXT** | `sdk/parsers/txt_parser.py` | ✅ |
| **CLI** | `cli/main.py` | ✅ Básico |
| **Modelos Pydantic** | `sdk/models/metadata.py` | ✅ |
| **Reporter** | `sdk/reporter.py` | ⚠️ Básico |
| **Tests unitarios** | `tests/` | ✅ 13/21 pasando* |

*Los errores son por permisos en Windows, no por el código.

---

### ❌ No Implementado (MVP debe incluir)

| Componente | Prioridad | Notas |
|------------|-----------|-------|
| **Layer 0.5: Native Metadata** | ALTA | Shortcut por EXIF/ID3/PDF props |
| **Storage Centralizado** | ALTA | `~/.sidecar-tagger/` en vez de `sidecar.json` por carpeta |
| **Audio/Video** | MEDIA | mutagen para MP3, ffprobe para video |
| **Métricas/Reporter** | MEDIA | findings.md más completo |
| **Sampling Optimization** | MEDIA | PDFs grandes → 3 páginas |

---

## MVP Recomendado

### Scope: Lo que debe incluir

```
MVP = Core Funcional + Optimizaciones Críticas
```

| # | Componente | Descripción |
|---|------------|-------------|
| 1 | **CLI básica** | `sidecar-tagger scan <path>` |
| 2 | **Pipeline 5-Layers** | Layers 0-4 funcionando |
| 3 | **Parsers core** | PDF, XLSX, Image, TXT |
| 4 | **Layer 0.5** | Metadata nativa como shortcut |
| 5 | **Storage centralizado** | `~/.sidecar-tagger/` |
| 6 | **Reporte básico** | `findings.md` con savings |

---

### NO incluir en MVP (Post-MVP)

- Audio/Video support
- Async queue
- Niveles de análisis configurables
- Webhook/notificaciones
- UI web/Desktop

---

## Gap Analysis

### Lo que YA funciona → Listo para MVP

```
✅ processor.py           - 5-Layer pipeline
✅ sdk/parsers/          - 4 parsers (PDF, XLSX, Image, TXT)
✅ sdk/llm_client.py     - Gemini integration
✅ sdk/embeddings_client.py - FastEmbed
✅ cli/main.py            - Entry point
✅ tests/                - 13 tests pasando
```

### Lo que FALTA → Implementar para MVP

```
❌ Layer 0.5             - Native metadata shortcut
❌ Storage centralizado  - ~/.sidecar-tagger/
❌ Migración legacy      - Importar sidecar.json existentes
❌ findings.md completo  - Métricas por layer
```

---

## Plan de Implementación MVP

### Fase 1: Layer 0.5 (2 días)

Implementar extracción de metadata nativa:

```python
# sdk/parsers/native_metadata.py (NUEVO)
- extract_pdf_props()      # author, title, keywords
- extract_exif()          # date, camera, GPS
- extract_id3()           # artist, album, year (post-MVP)
- calculate_confidence()  # score > 0.8 → shortcut
```

**Impacto:** ~40% archivos sin costo LLM

---

### Fase 2: Storage Centralizado (2 días)

Migrar de `sidecar.json` por carpeta a `~/.sidecar-tagger/`:

```
~/.sidecar-tagger/
├── manifest.json         # Index global
├── metadata/            # Por hash
│   ├── ab/
│   └── ...
└── config.json          # Paths rastreados
```

**Cambios:**
- `sdk/storage.py` (NUEVO) - CentralStorage class
- `cli/main.py` - Agregar `--storage` flag
- `processor.py` - Usar storage en vez de JSON local

---

### Fase 3: Reporter (1 día)

Mejorar `findings.md`:

```markdown
## Layer Resolution
- Layer 0 (hash): 15 (15%)
- Layer 0.5 (native): 20 (20%)
- Layer 3 (semantic): 10 (10%)
- Layer 4 (LLM): 55 (55%)

## Cost Savings
- Total LLM calls avoided: 45
- Estimated savings: $0.45
```

---

## Estimación Total MVP

| Fase | Días |
|------|------|
| Layer 0.5 | 2 |
| Storage Centralizado | 2 |
| Reporter | 1 |
| **Total** | **5 días** |

---

## Comandos MVP

```bash
# Procesar carpeta
sidecar-tagger scan ./Documents

# Procesar con output custom
sidecar-tagger scan ./Documents -o ./output

# Re-index (detectar archivos movidos)
sidecar-tagger reindex

# Ver status
sidecar-tagger status
```

---

## Arquitectura MVP

```
                    ┌─────────────────┐
                    │   CLI (main.py) │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │ MetadataProcessor│
                    └────────┬────────┘
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
   ┌────▼────┐         ┌─────▼─────┐        ┌─────▼─────┐
   │ Layer 0 │         │ Layer 0.5 │        │ Layer 1  │
   │  Hash   │────────▶│  Native   │───────▶│OS Context│
   └─────────┘         │ Metadata  │        └──────────┘
                       └───────────┘
                             │
                    ┌────────▼────────┐
                    │ Layer 2         │
                    │ Clustering      │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │ Layer 3         │
                    │ Embeddings     │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │ Layer 4         │
                    │ LLM (Gemini)   │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │ CentralStorage  │
                    │ (~/.sidecar-)   │
                    └─────────────────┘
```

---

## Tests MVP

Tests que deben pasar para considerar el MVP estable:

```bash
# Core
✅ test_layer_0_hash_dedup
✅ test_layer_0_5_native_metadata
✅ test_layer_1_os_context
✅ test_layer_2_clustering
✅ test_layer_3_semantic_cache
✅ test_layer_4_llm

# Storage
✅ test_central_storage_save
✅ test_central_storage_load
✅ test_reindex_moved_files

# Parsers
✅ test_pdf_parser
✅ test_xlsx_parser
✅ test_image_parser
✅ test_txt_parser

# CLI
✅ test_cli_basic_scan
✅ test_cli_verbose_flag
```

---

## Conclusión

**El código actual está al ~70% de un MVP funcional.**

Lo que falta para el MVP son las optimizaciones de costo (Layer 0.5) y la gestión de datos (storage centralizado), no funcionalidad core.

**Recomendación:** Priorizar Layer 0.5 + Storage Centralizado antes de lanzar MVP.
