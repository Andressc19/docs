# Extensiones de Archivo para MVP - Recomendaciones por Popularidad

> Análisis basado en estadísticas de uso real y relevancia para metadata tagging
> Fecha: 2026-03-30

---

## Resumen Ejecutivo

Basado en estadísticas de popularidad de archivos y el contexto del proyecto sidecar-tagger, este documento define las extensiones prioritarias para el MVP.

---

## Extensiones por Prioridad

### 🔴 PRIORIDAD ALTA (MVP Core)

| Extensión | Categoría | Popularidad | Parser Actual | Metadata Nativa |
|-----------|-----------|-------------|---------------|-----------------|
| **PDF 👌** | Documento | #64 | ✅ pdf_parser.py | EXIF/props |
| **DOCX 👌** | Documento | #48 | ❌ | Title, Author, Tags |
| **XLSX 👌** | Spreadsheet | #28 | ✅ xlsx_parser.py | Title, Author |
| **PNG 👌** | Imagen | #80 | ✅ image_parser.py | EXIF |
| **JPG/JPEG 👌** | Imagen | N/A (común) | ✅ image_parser.py | EXIF, GPS |
| **TXT ** | Texto | N/A (común) | ✅ txt_parser.py | N/A |
| **JSON 👌** | Texto | #12 | ❌ | N/A |

### 🟡 PRIORIDAD MEDIA (Post-MVP)

| Extensión | Categoría | Popularidad | Parser | Metadata Nativa |
|-----------|-----------|-------------|--------|-----------------|
| **DOC** | Documento | #26 | ❌ | Title, Author |
| **ODT** | Documento | #40 | ❌ | Title, Author |
| **XLS** | Spreadsheet | #95 | ❌ | Title, Author |
| **ODS** | Spreadsheet | #15 | ❌ | Title, Author |
| **PPTX** | Presentación | #97 | ❌ | Title, Author |
| **MP3** | Audio | N/A | ❌ (issue #03) | ID3 tags |
| **MP4** | Video | #30 | ❌ (issue #03) | metadata |
| **MKV** | Video | #9 | ❌ (issue #03) | metadata |

### 🟢 PRIORIDAD BAJA

| Extensión | Notas |
|-----------|-------|
| EPUB | #11 - E-books |
| CSV | Datos tabulares simples |
| MD | #94 - Markdown |
| XML | Estructurado |
| ZIP/RAR/7Z | Archivos comprimidos |
| DWG/DXF | CAD - complejidad alta |

---

## MVP vs Post-MVP

### Incluir en MVP (Core)

```
Documentos:
├── .pdf    (#64) - Ya implementado
├── .docx   (#48) - Alta demanda
└── .txt    -通用

Imágenes:
├── .png    (#80)
├── .jpg    / .jpeg
├── .gif    - Común
└── .webp   - Web

Spreadsheets:
└── .xlsx   (#28) - Ya implementado
```

### Excluir del MVP (Post-MVP)

```
Audio/Video:
├── .mp3, .flac, .wav, .m4a (Audio)
├── .mp4, .mkv, .avi, .mov (Video)

Documentos Legacy:
├── .doc, .odt, .pages

Presentaciones:
├── .pptx, .odp, .key

Comprimidos:
├── .zip, .rar, .7z

E-books:
├── .epub, .azw3, .fb2
```

---

## Recomendación de Scope MVP

Basado en la popularidad y complejidad de implementación:

| # | Extensión | Justificación |
|---|-----------|---------------|
| 1 | **PDF** | #64, ya funciona |
| 2 | **DOCX** | #48, requiere parser |
| 3 | **XLSX** | #28, ya funciona |
| 4 | **PNG/JPG** | Imágenes comunes |
| 5 | **TXT** | Universal |
| 6 | **JSON** | #12, datos estructurados |

**Extensiones objetivo MVP: ~6-8 formatos**

---

## Parsers a Implementar

### MVP

| Parser | Archivo | Estado | Complejidad |
|--------|---------|--------|-------------|
| PDF | `pdf_parser.py` | ✅ | Media |
| DOCX | `docx_parser.py` | ❌ NUEVO | Media |
| XLSX | `xlsx_parser.py` | ✅ | Media |
| Image | `image_parser.py` | ✅ | Baja |
| TXT | `txt_parser.py` | ✅ | Baja |
| JSON | `json_parser.py` | ❌ NUEVO | Baja |

### Post-MVP

| Parser | Justificación |
|--------|---------------|
| Audio (.mp3, .flac) | Issue #03 ya documentado |
| Video (.mp4, .mkv) | Issue #03 ya documentado |
| Legacy (.doc, .odt) | Formatos en declive |
| CSV | Datos simples, sin metadata |

---

## Justificación de Prioridades

### Por qué DOCX > PDF en prioridad para MVP?

1. **DOCX** (#48) es más común en entornos laborales donde el tagging es más valioso
2. **PDF** (#64) ya está implementado
3. **DOCX** contiene metadata estructurada (Author, Title, Tags, Subject)

### Por qué excluir Audio/Video del MVP?

1. **Complejidad**: Requiere ffprobe (external dependency)
2. **Metadata**: ID3/EXIF es más simple pero el caso de uso es diferente
3. **Issue #03** ya lo documenta para post-MVP
4. **Prioridad del negocio**: Documentos > Media personal

### Por qué incluir JSON?

1. **#12 en popularidad** - extremadamente común
2. **Estructura conocida** - fácil parseo
3. **Alto valor**: archivos de configuración, exports, APIs

---

## Referencias

- [File Extension Popularity Stats](https://www.file-extension.info/top)
- [MVP.md](./MVP.md) - Estado actual del proyecto
- [Issue #03](./issues/03-audio-video-support.md) - Audio/Video planificado

---

## Notas

- Las estadísticas de popularidad reflejan uso general, no específicamente el caso de uso de metadata tagging
- Extensiones como `.tmp`, `.dll`, `.exe` fueron excluidas por no ser relevantes para el caso de uso
- Prioridad final debe ajustarse según el usuario objetivo del MVP
