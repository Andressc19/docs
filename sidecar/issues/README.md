# Issues - sidecar-tagger

> Documentos de implementación basados en `../optimization-strategies.md`

---

## Índice

| # | Issue | Prioridad | Estado | Estimación |
|---|-------|-----------|--------|------------|
| 01 | [Layer 0.5 - Metadata Nativa](01-layer-0-5-native-metadata.md) | Alta | Pendiente | 3-5 días |
| 02 | [Storage Centralizado](02-centralized-storage.md) | Alta | Pendiente | 2-3 días |
| 03 | [Audio/Video Support](03-audio-video-support.md) | Media | Pendiente | 2 días |
| 04 | [Optimizar Sampling](04-optimize-sampling.md) | Media | Pendiente | 1 día |
| 05 | [Métricas y Reporter](05-metrics-reporter.md) | Media | Pendiente | 1-2 días |
| 06 | [Async Queue](06-async-queue.md) | Baja | Pendiente | 3-4 días |
| 07 | [Niveles de Análisis](07-analysis-levels.md) | Baja | Pendiente | 1-2 días |

---

## Roadmap Sugerido

### Sprint 1 (5-7 días)
- Issue 01: Layer 0.5 (native metadata)
- Issue 03: Audio/video support

### Sprint 2 (3-4 días)
- Issue 02: Centralized storage
- Issue 05: Métricas y reporter

### Sprint 3 (2-3 días)
- Issue 04: Optimización sampling

### Sprint 4 (4-6 días)
- Issue 06: Async queue (opcional)
- Issue 07: Niveles de análisis (opcional)

---

## Métricas de Éxito

| Métrica | Target | Actual |
|---------|--------|--------|
| % Layer 0 (hash) | >30% | ✅ |
| % Layer 0.5 (native) | >40% | 🔲 |
| % Layer 3 (semantic) | >20% | ✅ |
| Costo promedio/archivo | <$0.005 | ~$0.01 |
| Tiempo promedio/archivo | <500ms | ~1500ms |
