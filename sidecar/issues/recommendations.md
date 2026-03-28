# Issues Recomendados para Implementación

> Basado en: `../optimization-strategies.md`
> Fecha: 2026-03-27

---

## Prioridad Alta

### Issue 1: Implementar Layer 0.5 - Metadata Nativa como Shortcut

**Descripción:**
Agregar una capa entre Layer 0 (hash) y Layer 1 que extraiga metadata nativa del archivo (EXIF, ID3, PDF props) y haga shortcut si la confianza es alta (>0.8).

**Objetivo:**
- ~40% de archivos procesados sin costo API
- Latencia <10ms para archivos con metadata nativa

**Detalles técnicos:**
- PDF: `pypdf` o `pdfplumber` → author, title, subject, keywords
- Imágenes: `Pillow` EXIF → date, camera, GPS, orientation
- Audio: `mutagen` → ID3 tags (artist, album, title, year, genre)
- Video: container metadata (title, duration)

**Criterio de confianza:**
| Campo presente | Score |
|----------------|-------|
| author + title | +0.5 |
| keywords/subject | +0.2 |
| creation_date | +0.2 |
| subject | +0.1 |

**Threshold:** Si score > 0.8 → skip Layers 1-4

**Estimación:** 3-5 días de implementación

---

### Issue 2: Migrar storage a ubicación centralizada

**Descripción:**
Centralizar todos los datos en `~/.sidecar-tagger/` en lugar de generar `sidecar.json` en cada carpeta.

**Estructura propuesta:**
```
~/.sidecar-tagger/
├── config.json              # API keys, paths rastreados
├── manifest.json            # Index global (path → metadata summary)
├── metadata/                # Metadata detallada por hash
│   ├── ab/
│   │   └── abc123def456.json
│   └── ...
└── cache/
    └── embeddings/          # Embeddings (opcional)
```

**Features incluidos:**
- Comando `--reindex` para detectar archivos movidos
- Limpieza de orphans (paths obsoletos)
- Migración de archivos sidecar.json existentes

**Estimación:** 2-3 días

---

## Prioridad Media

### Issue 3: Agregar soporte audio/video

**Descripción:**
Extender parsers para soportar formatos de audio y video, integrando con Layer 0.5.

**Formatos soportados:**
- Audio: MP3, FLAC, WAV, M4A → mutagen
- Video: MP4, MKV, AVI → ffprobe (metadata container)

**Metadata a extraer:**
- Audio: artist, album, title, year, genre, duration
- Video: title, duration, codec, resolution

**Estimación:** 2 días

---

### Issue 4: Optimizar sampling para archivos grandes

**Descripción:**
Limitar la cantidad de contenido enviado al LLM para archivos grandes.

**Estrategias:**
- **PDFs:** Muestrear 3 páginas (inicio, medio, fin) en lugar de texto completo
- **Excel:** Solo primeras 5 filas de cada sheet
- **Imágenes:** Thumbnails en lugar de imagen original

**Beneficio:** Reducir tokens y latencia

**Estimación:** 1 día

---

### Issue 5: Sistema de métricas y reporter

**Descripción:**
Agregar tracking de qué layers resuelven cada archivo y generar reporte post-proceso.

**Métricas a tracking:**
- % resuelto por Layer 0 (hash dedup)
- % resuelto por Layer 0.5 (native metadata)
- % resuelto por Layer 3 (semantic cache)
- % resuelto por Layer 4 (LLM)
- Costo total vs archivos procesados

**Reporte `findings.md`:**
- Duplicate savings
- Cluster insights
- Anomalies detected

**Estimación:** 1-2 días

---

## Prioridad Baja

### Issue 6: Análisis diferido (async queue)

**Descripción:**
Procesar archivos en background queue, no bloquear espera de LLM.

**Trade-off:** No hay metadata inmediata, pero no bloquea UI

---

### Issue 7: Niveles de análisis configurables

**Descripción:**
Permitir configurar nivel de análisis.

| Nivel | Método | Costo |
|-------|--------|-------|
| fast | metadata nativa | $0 |
| standard | + embeddings cache | $0 |
| deep | + LLM | $ |

---

## Métricas de Éxito

| Métrica | Target | Actual |
|---------|--------|--------|
| % Layer 0 (hash) | >30% | ✅ Implementado |
| % Layer 3 (semantic) | >20% | ✅ Implementado |
| % Layer 0.5 (native) | >40% | 🔲 Pendiente |
| Costo promedio/archivo | <$0.005 | ~$0.01 |
| Tiempo promedio/archivo | <500ms | ~1500ms |

---

## Roadmap Sugerido

```
Sprint 1:
  - Issue 1: Layer 0.5 (native metadata)
  - Issue 3: Audio/video support

Sprint 2:
  - Issue 2: Centralized storage
  - Issue 5: Métricas y reporter

Sprint 3:
  - Issue 4: Optimización sampling
  - Issue 6-7: Features opcionales
```
