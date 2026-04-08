# SPEC: CLI Interface

**Documento:** Interface de línea de comandos para fileKor  
**Referencia:** INIT_SPEC.md sección 2

---

## Comandos Principales

| Comando | Descripción |
|--------|-------------|
| `filekor extract <path>` | Extrae texto del archivo |
| `filekor summarize <path>` | Genera resumen (short por defecto) |
| `filekor labels <path>` | Propone etiquetas |
| `filekor preview <path>` | Genera preview |
| `filekor sidecar <path>` | Genera archivo sidecar JSON |
| `filekor batch <directory>` | Procesa directorio completo |

---

## Uso Detallado

### `filekor extract`

```bash
filekor extract <path> [OPTIONS]

# Extrae texto de un PDF
filekor extract documento.pdf

# Extrae a archivo específico
filekor extract documento.pdf --output texto.txt

# Formato de salida
filekor extract documento.pdf --format json
```

**Options:**

| Flag | Descripción | Default |
|------|-------------|---------|
| `--output, -o` | Archivo de salida | stdout |
| `--format, -f` | Formato: text, json | text |
| `--cache` | Usar caché si existe | False |
| `--no-cache` | Forzar reprocesamiento | False |

---

### `filekor summarize`

```bash
filekor summarize <path> [OPTIONS]

# Resumen corto (default)
filekor summarize documento.pdf

# Resumen largo
filekor summarize documento.pdf --type long
filekor summarize documento.pdf -t long

# Guardar en archivo
filekor summarize documento.pdf --output resumen.txt
```

**Options:**

| Flag | Descripción | Default |
|------|-------------|---------|
| `--type, -t` | Tipo: short, long | short |
| `--output, -o` | Archivo de salida | stdout |

---

### `filekor labels`

```bash
filekor labels <path> [OPTIONS]

# Sugerir etiquetas
filekor labels documento.pdf

# Con confidence scores
filekor labels documento.pdf --show-confidence
```

**Output ejemplo:**

```
Suggested labels: contract, finance
Confidence:
  contract: 0.92
  finance: 0.88
  provider: 0.30
```

---

### `filekor preview`

```bash
filekor preview <path> [OPTIONS]

# Preview de archivo
filekor preview documento.pdf

# Preview con metadata
filekor preview documento.pdf --show-metadata

# Líneas limitadas
filekor preview documento.pdf --lines 50
```

**Output ejemplo:**

```
File: contrato-proveedor-2026.pdf
Size: 120 KB | Pages: 12 | Lang: es
----------------------------------------
[Preview]
Este contrato de servicios establece los términos y 
condiciones para la provisión de servicios cloud...
----------------------------------------
Labels: contract, provider
Summary: Commercial annex with provider pricing...
```

---

### `filekor sidecar`

```bash
filekor sidecar <path> [OPTIONS]

# Generar sidecar
filekor sidecar documento.pdf

# Directorio de salida
filekor sidecar documento.pdf --output ./metadata/

# Forzar regeneration
filekor sidecar documento.pdf --no-cache
```

**Output:**

Genera `documento.kor` en el mismo directorio o en `--output`.

---

### `filekor batch`

```bash
filekor batch <directory> [OPTIONS]

# Procesar directorio completo
filekor batch ./documentos/

# Nivel de análisis
filekor batch ./documentos/ --level fast

# Solo archivos nuevos/modificados
filekor batch ./documentos/ --incremental
```

**Options:**

| Flag | Descripción | Default |
|------|-------------|---------|
| `--level, -l` | Nivel: minimal, fast, standard, deep | standard |
| `--output, -o` | Directorio de salida | "." |
| `--incremental` | Solo procesar cambios | False |
| `--workers, -w` | Workers paralelos | 1 |

---

## Flags Globales

| Flag | Descripción | Default |
|------|-------------|---------|
| `--help, -h` | Mostrar ayuda | - |
| `--version, -v` | Mostrar versión | - |
| `--verbose` | Output detallado | False |
| `--quiet, -q` | Suprimir output | False |
| `--config` | Archivo de config | ~/.filekor.yaml |
| `--cache-dir` | Directorio de caché | ~/.filekor/cache |

---

## Config File

```yaml
# ~/.filekor.yaml
filekor:
  version: "1.0"
  
  cache:
    dir: "~/.filekor/cache"
    ttl_days: 30
    
  labels:
    taxonomy: default
    confidence_threshold: 0.7
    
  layers:
    level: standard
    layer_1_threshold: 0.8
    layer_2_threshold: 0.9
    
  llm:
    provider: gemini
    api_key: env:GEMINI_API_KEY
```

---

## Niveles de Análisis (CLI)

```bash
# Solo hash (dup detection)
filekor batch ./docs --level minimal

# Metadata rápida
filekor batch ./docs --level fast

# Con embeddings
filekor batch ./docs --level standard

# Full LLM
filekor batch ./docs --level deep
```

---

## Ejemplos de Uso

```bash
# 1. Escanear directorio por primera vez
filekor batch ./proyecto/ --level standard

# 2. Solo revisar archivos nuevos
filekor batch ./proyecto/ --incremental

# 3. Preview rápido de un archivo
filekor preview ./proyecto/contrato.pdf

# 4. Generar todos los sidecars
filekor batch ./proyecto/ --output ./proyecto/.filekor/

# 5. Resumen largo bajo demanda
filekor summarize ./proyecto/doc.pdf --type long
```

---

## Códigos de Salida

| Código | Significado |
|--------|-------------|
| 0 | Éxito |
| 1 | Error general |
| 2 | Error de argumentos |
| 3 | Archivo no encontrado |
| 4 | Caché corrupto |
| 5 | Error de permisos |
