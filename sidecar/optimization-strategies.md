# Sidecar-tagger: Documento de Diseño Técnico

> Este documento especifica la arquitectura, optimización y gestión de datos de sidecar-tagger.
> Está diseñado para que un agente IA pueda leerlo y explicar el sistema al usuario.

---

## 1. Propósito del Sistema

Sidecar-tagger es un motor de metadatos contextuales que genera archivos JSON sidecar con metadata semántica para archivos del sistema.

**Características principales:**
- Analiza contenido de archivos (PDF, imágenes, texto) usando IA
- NO modifica los archivos originales
- Genera metadata estructurada: doc_type, domain, tags, context
- Enfoque: precision > velocidad, accuracy > costo

---

## 2. Problema: Costo de API LLM

**Problema:** Cada archivo procesado requiere una llamada a Gemini API, lo cual tiene costo monetario y latencia.

**Impacto:**
- Costo por archivo: ~$0.01 (Gemini Flash)
- Tiempo por archivo: 1-3 segundos (latencia de red)
- Para 10,000 archivos: $100+ y horas de procesamiento

---

## 3. Arquitectura de Optimización de Costo (Pipeline de 5 Layers)

El sistema procesa cada archivo a través de 5 capas secuenciales. Cada capa puede "resolver" el archivo sin pasar a la siguiente, ahorrando costo.

### Layer 0: Binary Identity (Hash Deduplication)

**Función:** Detectar archivos exactamente duplicados usando SHA-256.

**Flujo:**
```
1. Calcular SHA-256 del archivo
2. Buscar hash en manifest existente
3. Si existe:
   - Clonar metadata del original (doc_type, tags, context)
   - Actualizar solo local_context (fechas actuales, path nuevo)
   - Marcar como "duplicate_of": <path_original>
   - Costo: $0 (no hay llamada a API)
4. Si no existe: Continuar a Layer 1
```

**Estado:** ✅ Implementado
**Impacto:** Archivos duplicados exactamente cuestan $0

---

### Layer 0.5: Metadata Nativa como Shortcut (PROPUESTA)

**Función:** Usar metadata embebida en el archivo como shortcut antes de enviar al LLM.

**¿Qué es la metadata nativa?** Datos embebidos por el creador/productor del archivo:

| Tipo | Metadata disponible | Ejemplo |
|------|---------------------|---------|
| PDF | Author, Title, Subject, Creator, Keywords | "Juan Pérez", "Informe Q1" |
| DOCX | author, title, subject, keywords | Mismo que PDF |
| XLSX | title, author, subject, comments | "Presupuesto 2024" |
| JPG/PNG | EXIF (fecha, cámara, GPS, orientación) | "2024-03-15", "iPhone 15 Pro" |
| MP3 | ID3 (artist, album, title, year, genre) | "The Beatles", "Abbey Road" |
| MP4 | container metadata (title, genre, duration) | "Documental Historia" |

**Flujo propuesto:**
```
1. Extraer metadata nativa del archivo
2. Calcular "metadata_confidence_score"
   - Si PDF tiene Author + Title填充 → score > 0.8
   - Si JPG tiene EXIF completo → score > 0.7
   - Si MP3 tiene ID3 tags → score > 0.8
3. Si score > threshold (0.8):
   - Construir FileMetadata desde metadata nativa
   - doc_type = inferred from metadata (PDF → "document", JPG → "image")
   - tags = from keywords/subject/EXIF
   - content_date = from creation date
   - Skip Layers 1-4 (sin costo LLM)
4. Si score < threshold: Continuar a Layer 1
```

**Implementación sugerida:**

```python
def extract_native_metadata(file_path: str) -> Optional[dict]:
    """Extrae metadata nativa según tipo de archivo."""
    ext = Path(file_path).suffix.lower()
    
    if ext == '.pdf':
        return extract_pdf_metadata(file_path)  # pypdf o pdfplumber
    elif ext in ['.jpg', '.jpeg', '.png', '.webp']:
        return extract_exif(file_path)  # Pillow EXIF
    elif ext in ['.mp3', '.flac']:
        return extract_id3(file_path)  # mutagen
    # ... otros tipos
    
def calculate_metadata_confidence(metadata: dict) -> float:
    """Calcula qué tan confiable es la metadata."""
    score = 0.0
    
    if metadata.get('author') and metadata.get('title'):
        score += 0.5
    if metadata.get('keywords'):
        score += 0.2
    if metadata.get('creation_date'):
        score += 0.2
    if metadata.get('subject'):
        score += 0.1
    
    return score  # 0.0 a 1.0

def check_metadata_shortcut(file_path: str) -> Optional[FileMetadata]:
    """Layer 0.5 principal."""
    metadata = extract_native_metadata(file_path)
    
    if metadata and calculate_metadata_confidence(metadata) > 0.8:
        return build_metadata_from_native(metadata)
    
    return None  # No hay shortcut
```

**Trade-off:**

| Pros | Contras |
|------|---------|
| Costo $0 (sin API) | Metadata puede ser incompleta o vacía |
| Instantáneo (<10ms) | No captura contenido semántico real |
| Datos objetivos (no hallucinations) | Creator puede haber puesto datos falsos |

---

### Layer 1: Context Enrichment (OS Context)

**Función:** Extraer datos objetivos del sistema operativo.

**Datos extraídos:**
- filename: nombre del archivo
- file_extension: extensión (.pdf, .jpg, etc.)
- file_size_bytes: tamaño en bytes
- creation_date: fecha de creación (ISO 8601)
- modification_date: fecha de modificación
- owner: propietario del archivo
- parent_folder: carpeta padre
- path_keywords: palabras de la ruta (ej: "Projects/2024/Invoice")

**Estado:** ✅ Implementado

---

### Layer 2: Collective Intelligence (Clustering)

**Función:** Agrupar archivos por similitud de nombre en la misma carpeta.

**Flujo:**
```
1. Agrupar archivos por directorio padre
2. Aplicar fuzzy matching (difflib.SequenceMatcher)
3. Si similarity > 0.6 → mismo cluster
4. Generar ClusterHint para cada archivo
   - cluster_id: identificador del cluster
   - similarity_score: similitud con líder
   - is_anomaly: true si no encaja con vecinos
5. Usar hints como contexto para LLM
```

**Estado:** ✅ Implementado

---

### Layer 3: Semantic Identity (Embeddings Cache)

**Función:** Usar embeddings locales para detectar contenido similar.

**Flujo:**
```
1. Generar embedding del contenido extraído (FastEmbed ONNX)
2. Comparar con embeddings en manifest (cosine similarity)
3. Si similarity > 0.9:
   - Reutilizar metadata del archivo similar
   - Actualizar solo local_context
   - Costo: $0 (FastEmbed local)
4. Si no hay match: Continuar a Layer 4
```

**Estado:** ✅ Implementado
**Impacto:** Contenido semánticamente similar no necesita LLM

---

### Layer 4: Cognitive Analysis (LLM)

**Función:** Análisis profundo con Gemini API.

**Último recurso** cuando Layers 0-3 no resoluvieron.

**Flujo:**
```
1. Construir prompt con:
   - Context block (Layer 1): OS facts objetivos
   - Hint block (Layer 2): cluster context
   - Content: texto extraído o imagen
2. Enviar a Gemini API
3. Parsear respuesta JSON → FileMetadata
4. Guardar en manifest con embedding
```

**Costo:** ~$0.01 por archivo

---

## 4. Gestión de Archivos de Datos

### Problema: Dónde almacenar los datos

El sistema genera archivos de datos que deben persistirse entre ejecuciones.

**Opciones evaluadas:**

| Opción | Descripción | Ventajas | Contras |
|--------|-------------|----------|---------|
| A. Por carpeta | `carpeta/sidecar.json` | Cercano al contenido | "Mugre" en cada carpeta |
| B. Centralizado | Todo en `~/.sidecar-tagger/` | Sin archivos en carpetas | No sobrevive moves |
| C. Subcarpeta oculta | `carpeta/.sidecar/data.json` | Ordenado, no visible | Más path drilling |

### Decisión: Ubicación Centralizada

**Arquitectura seleccionada:**

```
~/.sidecar-tagger/              # Directorio base (home del usuario)
├── config.json                  # Configuración (API keys, paths rastreados)
├── manifest.json                # Index global (path → metadata summary)
├── metadata/                    # Metadata detallada por hash
│   ├── ab/                      # Prefijo del hash (sharding)
│   │   └── abc123def456.json    # Archivo individual por hash SHA-256
│   ├── cd/
│   │   └── ...
│   └── ...
└── cache/                       # Cache opcional
    └── embeddings/              # Embeddings (puede crecer mucho)
```

**Estructura de manifest.json:**

```json
{
  "version": "2.0",
  "last_updated": "2026-03-26T10:00:00Z",
  "index": {
    "/home/user/Documents/report.pdf": {
      "file_hash": "abc123...",
      "doc_type": "invoice",
      "domain": "finance",
      "tags": ["azure", "billing"],
      "last_indexed": "2026-03-26T10:00:00Z"
    }
  }
}
```

**Estructura de metadata/{hash}.json:**

```json
{
  "file_hash": "abc123...",
  "original_path": "/home/user/Documents/report.pdf",
  "doc_type": "invoice",
  "language": "es",
  "domain": "finance",
  "category": "accounts_payable",
  "context": "Invoice from Supplier X for cloud services",
  "tags": ["cloud", "azure", "billing"],
  "content_date": "2024-05-22T10:00:00Z",
  "confidence": 0.95,
  "needs_review": false,
  "local_context": { ... },
  "cluster_hint": { ... },
  "embedding_vector": [0.1, -0.2, ...]
}
```

### Gestión de Paths Obsoletos

**Problema:** Si el usuario mueve un archivo, el path en manifest queda obsolete.

**Solución:**

```
1. Re-index periódicamente (ej: daily/weekly)
2. Para cada path en manifest:
   a. Verificar si archivo existe en path guardado
   b. Si no existe:
      - Buscar por hash en el filesystem
      - Si se encuentra, actualizar path
      - Si no, marcar como "orphan" (path obsoleto)
3. Limpiar orphans periódicamente
```

**Comando propuesto:**

```bash
sidecar-tagger --reindex --scan-paths /home/user/Documents,/home/user/Downloads
```

---

## 5. Estrategias Adicionales (Futuro)

### 5.1 Skip por Confianza Heurística

```
Si filename sugiere contenido confiable:
  - "invoice_azure_2024.pdf" → probablemente es invoice
  - Análisis opcional, no obligatorio
```

**Trade-off:** Menos preciso, más rápido

---

### 5.2 Análisis Diferido (Async Queue)

```
- Procesar archivos en background queue
- No esperar respuesta LLM inmediatamente
- Usuario puede usar metadata parcial mientras completa
```

**Trade-off:** No hay metadata inmediata, pero no bloquea

---

### 5.3 Niveles de Análisis

| Nivel | Método | Costo | Precisión |
|-------|--------|-------|-----------|
| 1 | Solo filename/path | $0 | Baja (enganchable) |
| 2 | Metadata nativa | $0 | Media |
| 3 | Embeddings (cache hit) | $0 | Alta |
| 4 | LLM | $ | Muy alta |

**Uso recomendado:**
- Pre-indexación rápida → nivel 1-2
- Precisión crítica → nivel 4

---

## 6. Recomendaciones de Implementación

### Prioridad 1: Mantener lo actual
- Layers 0 y 3 ya implementados son el mayor ahorro
- No cambiar lo que funciona

### Prioridad 2: Agregar Layer 0.5
- Metadata nativa es un shortcut de bajo esfuerzo
- Bajo riesgo, alto retorno
- ROI: ~40% de archivos podrían usar shortcut

### Prioridad 3: Migrar a ubicación central
- Implementar `~/.sidecar-tagger/` como storage
- Eliminar generación de sidecar.json en carpetas

### Prioridad 4: Optimizar sampling
- PDFs grandes: solo muestrear 3 páginas (inicio, medio, fin)
- Excel: solo primeras 5 filas de cada sheet

---

## 7. Métricas de Éxito

| Métrica | Target | Actual |
|---------|--------|--------|
| % de archivos con hash dedup (Layer 0) | >30% | Implementado |
| % de archivos con semantic cache (Layer 3) | >20% | Implementado |
| % de archivos con native metadata (Layer 0.5) | >40% | Pendiente |
| Costo promedio por archivo | <$0.005 | ~$0.01 |
| Tiempo promedio por archivo | <500ms | ~1500ms |

---

## 8. Glosario

| Término | Definición |
|---------|------------|
| **Sidecar** | Archivo JSON generado junto al archivo original |
| **Manifest** | Archivo index que mapea paths a metadata |
| **Hash dedup** | Deduplicación por contenido binario exacto |
| **Semantic cache** | Reutilizar metadata por similitud de contenido |
| **Metadata nativa** | Datos embebidos dentro del archivo (EXIF, ID3, PDF props) |
| **Layer** | Etapa del pipeline de procesamiento |

---

*Documento generado: 2026-03-26*
*Proyecto: sidecar-tagger v2*
*Este documento puede ser procesado por agentes IA para explicar el sistema*