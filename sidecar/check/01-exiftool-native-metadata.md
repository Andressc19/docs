# Issue 2: Layer 1 - Native + OS Metadata (EXIFTOOL)

**Estado:** Pendiente  
**Prioridad:** Alta  
**Estimación:** 2-3 días

> **Depende de:** Issue 1 (Análisis Levels + Tests)

---

## Descripción

**Layer 1** combina:
1. **Native Metadata**: Extraída via EXIFTOOL (metadata DEL ARCHIVO)
2. **OS Metadata**: Extraída del sistema (metadata DEL ARCHIVO)

Si confidence >= 0.8 → **shortcut** (no necesita Layers 2-3)

---

## Objetivo

- Extraer metadata nativa + OS sin costo API
- Soporte cross-platform (macOS, Windows, Linux)
- Latencia <50ms por archivo
- Formatos: PDF, DOCX, XLSX, PNG, JPG, JSON

---

## ¿Por qué EXIFTOOL?

| Opción | Pros | Contras |
|--------|------|---------|
| **EXIFTOOL** | Maduro, 25k+ formatos, cross-platform, CLI | External dependency |
| Implementar de cero | Control total | months de trabajo, bugs |

**Decisión:** Usar EXIFTOOL - es el estándar de facto para metadata.

---

## Metadata Layer 1

### Native (EXIFTOOL) - metadata DEL ARCHIVO

| Extensión | Categoría | Campos |
|-----------|-----------|--------|
| `.pdf` | Documento | Title, Author, Subject, Creator, Keywords |
| `.docx` | Documento | Title, Author, Subject, Keywords |
| `.xlsx` | Spreadsheet | Title, Author, Subject, Comments |
| `.png` | Imagen | date, camera_make, camera_model, GPS |
| `.jpg` / `.jpeg` | Imagen | date, camera_make, camera_model, GPS |
| `.txt` | Texto | N/A |
| `.json` | Datos | keys de primer nivel |

### OS - metadata DEL SISTEMA

| Campo | Descripción |
|-------|-------------|
| `filename` | Nombre del archivo |
| `path` | Path relativo desde raíz escaneada |
| `file_size` | Tamaño en bytes |
| `created` | Fecha de creación |
| `modified` | Fecha de modificación |
| `extension` | Extensión del archivo |

---

## Instalación

### Windows

```powershell
# Opción 1: Chocolatey
choco install exiftool

# Opción 2: Descargar .exe
# https://exiftool.org/
```

### macOS

```bash
# Homebrew
brew install exiftool
```

---

## Implementación

```python
# sdk/native_metadata/exiftool_client.py
import subprocess
import json
import os
from pathlib import Path
from typing import Optional

class ExifToolClient:
    EXIFTOOL_CMD = "exiftool"
    
    def __init__(self):
        self._verify_exiftool()
    
    def _verify_exiftool(self):
        """Verificar que exiftool esté instalado"""
        result = subprocess.run(
            [self.EXIFTOOL_CMD, "-ver"],
            capture_output=True,
            text=True
        )
        if result.returncode != 0:
            raise RuntimeError(
                "exiftool no está instalado. "
                "Instala: https://exiftool.org/"
            )
    
    def extract(self, file_path: str) -> Optional[dict]:
        """Extraer metadata como JSON"""
        cmd = [
            self.EXIFTOOL_CMD,
            "-j",  # Output JSON
            "-G",  # Group names
            file_path
        ]
        
        result = subprocess.run(
            cmd,
            capture_output=True,
            text=True,
            timeout=30
        )
        
        if result.returncode != 0:
            return None
        
        try:
            data = json.loads(result.stdout)
            return data[0] if data else None
        except json.JSONDecodeError:
            return None
    
    def extract_tags(self, file_path: str, tags: list[str]) -> Optional[dict]:
        """Extraer solo tags específicos"""
        cmd = [
            self.EXIFTOOL_CMD,
            "-j",
            "-G",
            *[f"-{tag}" for tag in tags],
            file_path
        ]
        
        result = subprocess.run(cmd, capture_output=True, text=True)
        
        if result.returncode != 0:
            return None
        
        try:
            data = json.loads(result.stdout)
            return data[0] if data else None
        except json.JSONDecodeError:
            return None
```

---

## Mapeo de Tags por Extensión

```python
# sdk/native_metadata/tag_mapper.py
from typing import Dict, List

TAG_MAPPING: Dict[str, Dict[str, str]] = {
    ".pdf": {
        "Title": "title",
        "Author": "author",
        "Subject": "subject",
        "Keywords": "keywords",
        "Creator": "creator",
    },
    ".docx": {
        "Title": "title",
        "Author": "author",
        "Subject": "subject",
        "Keywords": "keywords",
    },
    ".xlsx": {
        "Title": "title",
        "Author": "author",
        "Subject": "subject",
        "Comments": "comments",
    },
    ".png": {
        "DateTimeOriginal": "date",
        "Make": "camera_make",
        "Model": "camera_model",
        "GPSLatitude": "gps_lat",
        "GPSLongitude": "gps_lon",
    },
    ".jpg": {
        "DateTimeOriginal": "date",
        "Make": "camera_make",
        "Model": "camera_model",
        "GPSLatitude": "gps_lat",
        "GPSLongitude": "gps_lon",
    },
}

def map_tags(raw_metadata: dict, extension: str) -> dict:
    """Mapear tags crudos a formato interno"""
    mapping = TAG_MAPPING.get(extension.lower(), {})
    
    result = {}
    for exif_key, internal_key in mapping.items():
        # Buscar en diferentes grupos de EXIFTOOL
        for group in ["", "EXIF", "IPTC", "XMP"]:
            full_key = f"{group}:{exif_key}" if group else exif_key
            if full_key in raw_metadata:
                result[internal_key] = raw_metadata[full_key]
                break
    
    return result
```

---

## Score de Confianza

```python
# sdk/native_metadata/confidence.py
def calculate_confidence(metadata: dict) -> float:
    """Calcular score de confianza 0.0 - 1.0"""
    score = 0.0
    
    # Alta prioridad
    if metadata.get("author") and metadata.get("title"):
        score += 0.5
    elif metadata.get("author") or metadata.get("title"):
        score += 0.25
    
    # Media prioridad
    if metadata.get("keywords") or metadata.get("subject"):
        score += 0.2
    
    # Baja prioridad
    if metadata.get("date") or metadata.get("creation_date"):
        score += 0.2
    
    if metadata.get("camera_make") or metadata.get("camera_model"):
        score += 0.1
    
    return min(score, 1.0)

# Threshold para shortcut
CONFIDENCE_THRESHOLD = 0.8
```

---

## Implementación Layer 1

```python
# sdk/layers/layer_1.py
import os
import json
from pathlib import Path
from typing import Optional
from sdk.native_metadata.exiftool_client import ExifToolClient
from sdk.native_metadata.tag_mapper import map_tags
from sdk.native_metadata.confidence import calculate_confidence, CONFIDENCE_THRESHOLD
from sdk.models.metadata import FileMetadata

SUPPORTED_EXTENSIONS = {".pdf", ".docx", ".xlsx", ".png", ".jpg", ".jpeg", ".json"}

class Layer1:
    """Layer 1: Native + OS Metadata"""
    
    def __init__(self):
        self.exiftool = ExifToolClient()
    
    def process(self, file_path: str) -> Optional[FileMetadata]:
        """Extraer native + OS metadata"""
        ext = Path(file_path).suffix.lower()
        
        # Extraer OS metadata
        os_metadata = self._extract_os_metadata(file_path)
        
        # Extraer Native metadata (EXIFTOOL)
        native_metadata = {}
        if ext in SUPPORTED_EXTENSIONS:
            native_metadata = self._extract_native_metadata(file_path, ext)
        
        # Combinar
        combined = {**os_metadata, **native_metadata}
        
        # Calcular confidence
        confidence = calculate_confidence(native_metadata)
        
        # Shortcut si confidence >= 0.8
        if confidence >= CONFIDENCE_THRESHOLD:
            return self._build_metadata(combined, confidence, ext)
        
        return None  # Continuar a Layer 2
    
    def _extract_os_metadata(self, file_path: str) -> dict:
        """Extraer metadata del sistema operativo"""
        stat = os.stat(file_path)
        return {
            "filename": Path(file_path).name,
            "path": str(Path(file_path)),
            "file_size": stat.st_size,
            "created": stat.st_ctime,
            "modified": stat.st_mtime,
            "extension": Path(file_path).suffix.lower()
        }
    
    def _extract_native_metadata(self, file_path: str, ext: str) -> dict:
        """Extraer metadata nativa via EXIFTOOL"""
        raw = self.exiftool.extract(file_path)
        if not raw:
            return {}
        return map_tags(raw, ext)
    
    def _build_metadata(self, combined: dict, confidence: float, ext: str) -> FileMetadata:
        return FileMetadata(
            doc_type=self._infer_doc_type(ext),
            tags=self._extract_tags(combined),
            content_date=combined.get("date") or combined.get("modified"),
            source="layer_1",
            confidence=confidence,
            native_metadata=combined
        )
    
    def _infer_doc_type(self, ext: str) -> str:
        mapping = {
            ".pdf": "pdf_document",
            ".docx": "word_document",
            ".xlsx": "spreadsheet",
            ".png": "image",
            ".jpg": "image",
            ".jpeg": "image",
            ".json": "data",
        }
        return mapping.get(ext, "unknown")
    
    def _extract_tags(self, combined: dict) -> list[str]:
        tags = []
        
        # Keywords
        keywords = combined.get("keywords", "")
        if keywords:
            if isinstance(keywords, str):
                tags.extend([t.strip() for t in keywords.split(",")])
        
        # Author
        if combined.get("author"):
            tags.append(str(combined["author"]))
        
        # Camera
        camera = " ".join([
            combined.get("camera_make", ""),
            combined.get("camera_model", "")
        ]).strip()
        if camera:
            tags.append(camera)
        
        return list(set(tags))
```

---

## Tests Recomendados

- `test_exiftool_installed()` - Verificar instalación
- `test_layer_1_os_metadata()` - Extraer filename, path, size
- `test_layer_1_native_metadata()` - Extraer via EXIFTOOL
- `test_layer_1_shortcut()` - Confidence >= 0.8 → shortcut
- `test_layer_1_continue()` - Confidence < 0.8 → continua
- `test_confidence_calculation()` - Casos edge

---

## Métricas Esperadas

| Métrica | Target |
|---------|--------|
| Latencia por archivo | <50ms |
| % shortcut Layer 1 | ~40% |
| Formatos soportados | 7 |

---

## Dependencias del Sistema

```bash
# Windows
choco install exiftool

# macOS
brew install exiftool

# Linux
apt install libimage-exiftool-perl  # Debian/Ubuntu
yum install epel-release && yum install perl-Image-ExifTool  # RHEL/CentOS
```

---

## Referencias

- [EXIFTOOL](https://exiftool.org/)
- [file-extensions.md](../mvp/file-extensions.md)
- [MVP.md](../mvp/MVP.md)
