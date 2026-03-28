# Issue 1: Implementar Layer 0.5 - Metadata Nativa como Shortcut

**Estado:** Pendiente  
**Prioridad:** Alta  
**Estimación:** 3-5 días

---

## Descripción

Agregar una capa entre Layer 0 (hash) y Layer 1 que extraiga metadata nativa del archivo y haga shortcut si la confianza es alta (>0.8), evitando el costo del LLM.

---

## Objetivo

- ~40% de archivos procesados sin costo API
- Latencia <10ms para archivos con metadata nativa
- Reducir costo promedio por archivo de $0.01 a <$0.005

---

## Metadata Nativa por Tipo de Archivo

| Tipo | Herramienta | Campos a Extraer |
|------|-------------|------------------|
| PDF | pypdf/pdfplumber | author, title, subject, creator, keywords |
| DOCX | python-docx | author, title, subject, keywords |
| XLSX | openpyxl | title, author, subject, comments |
| JPG/PNG | Pillow (EXIF) | date, camera, GPS, orientation |
| MP3/FLAC | mutagen | artist, album, title, year, genre |
| MP4 | ffprobe | title, duration, codec |

---

## Criterio de Confianza

| Campo Presente | Score |
|----------------|-------|
| author + title | +0.5 |
| keywords/subject | +0.2 |
| creation_date | +0.2 |
| subject | +0.1 |

**Threshold:** Si score > 0.8 → skip Layers 1-4

---

## Flujo Propuesto

```
1. Extraer metadata nativa del archivo
2. Calcular metadata_confidence_score
3. Si score > 0.8:
   - Construir FileMetadata desde metadata nativa
   - doc_type = inferred from extension
   - tags = from keywords/subject/EXIF
   - content_date = from creation date
   - Skip Layers 1-4 (sin costo LLM)
4. Si score < 0.8: Continuar a Layer 1
```

---

## Pseudocódigo

```python
def extract_native_metadata(file_path: str) -> Optional[dict]:
    ext = Path(file_path).suffix.lower()
    
    if ext == '.pdf':
        return extract_pdf_metadata(file_path)
    elif ext in ['.jpg', '.jpeg', '.png', '.webp']:
        return extract_exif(file_path)
    elif ext in ['.mp3', '.flac']:
        return extract_id3(file_path)
    # ...

def calculate_metadata_confidence(metadata: dict) -> float:
    score = 0.0
    if metadata.get('author') and metadata.get('title'):
        score += 0.5
    if metadata.get('keywords'):
        score += 0.2
    if metadata.get('creation_date'):
        score += 0.2
    if metadata.get('subject'):
        score += 0.1
    return score

def layer_0_5_shortcut(file_path: str) -> Optional[FileMetadata]:
    metadata = extract_native_metadata(file_path)
    if metadata and calculate_metadata_confidence(metadata) > 0.8:
        return build_metadata_from_native(metadata)
    return None
```

---

## Dependencias a Agregar

```
# requirements.txt
mutagen>=1.47.0    # Audio metadata
Pillow>=10.0.0     # EXIF extraction
python-docx>=1.1.0 # DOCX metadata
openpyxl>=3.1.0    # XLSX metadata
ffmpeg-python>=0.2.0  # Video metadata (optional)
```

---

## Tests Recomendados

- `test_native_metadata_pdf()` - PDF con/sin metadata
- `test_native_metadata_image()` - JPG con/sin EXIF
- `test_native_metadata_audio()` - MP3 con/sin ID3
- `test_confidence_score_calculation()` - Casos edge
- `test_layer_0_5_integration()` - Integración con processor

---

## Métricas Esperadas

| Métrica | Antes | Después |
|---------|-------|---------|
| % resuelto por Layer 0.5 | 0% | ~40% |
| Costo promedio/archivo | $0.01 | <$0.005 |
| Latencia promedio | ~1500ms | <500ms |
