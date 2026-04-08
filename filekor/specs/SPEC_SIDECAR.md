# SPEC: Sidecar Schema

**Documento:** Schema del archivo sidecar JSON generado por fileKor  
**Referencia:** INIT_SPEC.md sección 5

---

## Propósito

El sidecar es un archivo JSON que acompaña a cada archivo procesado, permitiendo:
- **Portabilidad:** Si mueves archivos, la inteligencia viaja con ellos
- **Consumidor compatible:** Schema diseñado para consumidores (TUIs, pipelines)
- **Independencia:** Reconstrucción sin re-procesar

---

## Schema: `archivo.kor`

```json
{
  "version": "1.0",
  "file": {
    "path": "contracts/price-contract.pdf",
    "name": "price-contract.pdf",
    "extension": "pdf",
    "size_bytes": 120344,
    "modified_at": "2026-04-01T10:00:00Z",
    "hash_sha256": "abc123..."
  },
  "metadata": {
    "author": "John Doe",
    "created": "2026-03-15T09:30:00Z",
    "pages": 12
  },
  "content": {
    "language": "es",
    "word_count": 1542
  },
  "summary": {
    "short": "Commercial annex with provider pricing for Q2 2026...",
    "long": "Full analysis: This document contains a commercial annex..."
  },
  "labels": {
    "suggested": ["finance", "contract"],
    "confidence": {
      "finance": 0.92,
      "contract": 0.88
    }
  },
  "parser_status": "OK",
  "generated_at": "2026-04-08T10:00:00Z",
  "generated_by": "filekor"
}
```

---

## Campos Detallados

### `version` (string, requerido)

Versión del schema. Permite futuras migraciones.

### `file` (object, requerido)

Información básica del archivo.

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `path` | string | Ruta relativa al archivo |
| `name` | string | Nombre del archivo con extensión |
| `extension` | string | Extensión (ej: "pdf") |
| `size_bytes` | int | Tamaño en bytes |
| `modified_at` | string (ISO 8601) | Última modificación |
| `hash_sha256` | string | Hash para deduplicación |

### `metadata` (object, opcional)

Metadatos extraídos de exiftool/PyExifTool.

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `author` | string | Autor del documento |
| `created` | string | Fecha de creación |
| `pages` | int | Número de páginas (PDFs) |
| *(otros)* | mixed | Tags adicionales de exiftool |

### `content` (object, opcional)

Información del contenido extraído.

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `language` | string | Idioma detectado (ej: "es", "en") |
| `word_count` | int | Cantidad de palabras |
| `page_count` | int | Conteo de páginas |

### `summary` (object, opcional)

Resúmenes generados.

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `short` | string | Resumen corto (heurístico) |
| `long` | string | Resumen largo (análisis completo) |

### `labels` (object, opcional)

Etiquetas sugeridas.

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `suggested` | string[] | Lista de etiquetas propuestas |
| `confidence` | dict | Scores de confianza por etiqueta |

### `parser_status` (string, requerido)

Estado del procesamiento.

| Valor | Significado |
|-------|-------------|
| `OK` | Extraído correctamente |
| `DEGRADED` | Encriptado o parcial — `"Needs OCR or password"` |
| `BROKEN` | No se puede leer |

### `generated_at` (string, requerido)

Timestamp ISO 8601 de cuando se generó.

### `generated_by` (string, requerido)

Siempre `"filekor"`.

---

## Ubicación del Sidecar

| Opción | Ubicación | Pros | Contras |
|--------|-----------|------|---------|
| **A. Por archivo** | `archivo.kor` junto al archivo | Portabilidad máxima | "Mugre" en carpetas |
| **B. Centralizado** | `.filekor/sidecars/{hash}.kor` | Limpio | Se pierde al mover |
| **C. Subcarpeta** | `carpeta/.filekor/data.json` | Balance | Path drilling |

**Decisión:** Por defecto centralizado, opcional por archivo para portabilidad.

---

## Caché Relacionado

El sidecar puede referenciar archivos de caché:

```
.filekor/
├── sidecars/
│   └── {hash_sha256}.json
├── cache/
│   ├── text/
│   │   └── {hash_sha256}.txt      # Texto extraído
│   └── previews/
│       └── {hash_sha256}.txt      # Preview/snippet
```

---

## Ejemplo de Caché en Sidecar (futuro)

```json
{
  "version": "1.0",
  "file": { ... },
  "cache_refs": {
    "text": ".filekor/cache/text/abc123.txt",
    "preview": ".filekor/cache/previews/abc123.txt"
  },
  ...
}
```

---

## Validación

El sidecar debe ser válido contra el schema JSON. Usar Pydantic para validación:

```python
from pydantic import BaseModel, Field

class SidecarSchema(BaseModel):
    version: str = Field(default="1.0")
    file: FileInfo
    metadata: Optional[MetadataInfo] = None
    content: Optional[ContentInfo] = None
    summary: Optional[SummaryInfo] = None
    labels: Optional[LabelsInfo] = None
    parser_status: ParserStatus
    generated_at: datetime
    generated_by: Literal["filekor"]
```
