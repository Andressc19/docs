# fileKor — Preguntas de Integración con Consumidores

**Estado:** Pendiente de respuesta  
**Fecha:** 2026-04-08  
**Purpose:** Definir cómo los consumidores (TUIs, scripts, pipelines) consumirán fileKor durante desarrollo en paralelo

---

## 1. Modo de Integración

### Q1: ¿Cómo el consumidor debe detectar qué capacidades de fileKor están disponibles?

| Opción | Descripción |
|--------|-------------|
| **A. Auto-detect** | fileKor intenta cargar deps, usa mock si no están |
| **B. Config explícito** | Consumidor configura qué capas usar manualmente |
| **C. Versioning API** | fileKor expone versión + capacidades disponibles |

**Ejemplo A:**
```python
processor = MetadataProcessor()  # Auto-detecta
# Si PyExifTool no está → usa MockMetadataAdapter
```

**Ejemplo B:**
```python
filekor.configure(layers=["mock", "mock", "mock"])  # Consumidor decide
```

**Ejemplo C:**
```python
capabilities = filekor.get_capabilities()
# {"extract": "mock", "summarize": "real", "labels": "real"}
```

### Q2: ¿Qué funciones mínimas necesita el consumidor para funcionar?

| Función | ¿Mínimo requerido? |
|---------|---------------------|
| `extract()` | ¿Sí/No? |
| `summarize()` | ¿Sí/No? |
| `labels()` | ¿Sí/No? |
| `preview()` | ¿Sí/No? |
| `sidecar()` | ¿Sí/No? |

### Q3: ¿El mock debe responder con datos realistas o puede ser genérico?

| Opción | Descripción |
|--------|-------------|
| **Genérico** | Siempre retorna el mismo JSON mock |
| **Semi-realista** | Retorna datos basados en el filename/path |
| **Configurable** | Consumidor puede pasar sus propios datos de mock |

---

## 2. Mock Data

### Q4: ¿fileKor debe incluir mock data pre-configurado?

Ejemplo: PDF de contrato con metadata real para testing.

| Opción | Descripción |
|--------|-------------|
| **Sí, incluir** | `filekor fixtures/sample-contract.pdf` |
| **No, consumidor provee** | Consumidor pasa sus propios archivos de test |

### Q5: ¿Cómo el consumidor pasa datos de mock a fileKor?

```python
# Opción A: Config file
filekor.configure(mock_data_path="./test/fixtures/")

# Opción B: Direct API
filekor.set_mock_data({
    "extract": {"text": "Mock contract..."},
    "labels": {"suggested": ["contract"]}
})

# Opción C: Fixture discovery
# fileKor busca ./fixtures/ automáticamente
```

---

## 3. API Contract

### Q6: ¿Qué nivel de API necesita el consumidor?

| Nivel | Descripción |
|-------|-------------|
| **Basic** | Solo resultados (sin metadata de procesamiento) |
| **Detailed** | Resultados + metadata (qué capa se usó, confidence, etc.) |
| **Debug** | Todo + logs internos |

```python
# Basic
result = filekor.extract("doc.pdf")
# {"text": "...", "pages": 12}

# Detailed
result = filekor.extract("doc.pdf", verbose=True)
# {"text": "...", "pages": 12, "layer_used": "L1", "confidence": 0.85}

# Debug
result = filekor.extract("doc.pdf", debug=True)
# {"text": "...", "layer_used": "L1", "llm_calls": 0, "cache_hit": False}
```

---

## 4. Fallback Behavior

### Q7: Si una capa falla (ej: LLM no disponible), ¿qué debe pasar?

| Opción | Comportamiento |
|--------|----------------|
| **Fail fast** | Retorna error inmediatamente |
| **Graceful** | Retorna lo que pudo obtener de capas anteriores |
| **Configurable** | El consumidor decide el comportamiento |

```python
# Fail fast
try:
    result = processor.process("doc.pdf")
except LayerUnavailableError as e:
    raise e  # Consumidor maneja el error

# Graceful
result = processor.process("doc.pdf")
# Si L3 falla, retorna resultado de L1 con confidence parcial

# Configurable
filekor.configure(fallback="graceful")  # o "fail_fast"
```

---

## 5. Versioning

### Q8: ¿fileKor debe tener versionado de API?

| Opción | Descripción |
|--------|-------------|
| **Semver** | Major = breaking, Minor = new features, Patch = fixes |
| **API version** | `/v1/`, `/v2/` en imports |
| **Capabilities** | `filekor.get_capabilities()` retorna features disponibles |

### Q9: ¿Cuál es la versión mínima que el consumidor soporta?

Ejemplo: Consumidor soporta fileKor `>=0.2.0`

---

## 6. Instalación

### Q10: ¿Cómo el consumidor instala/consume fileKor?

| Opción | Descripción |
|--------|-------------|
| **PyPI** | `pip install filekor` |
| **Local path** | `pip install -e /path/to/filekor` |
| **Git submodule** | Repo como submodule |
| **Ambos** | PyPI para release, local para dev |

---

## Resumen de Preguntas

| # | Pregunta | Opciones |
|---|----------|----------|
| Q1 | Detección de capacidades | Auto/Config/Versioning |
| Q2 | Funciones mínimas | extract/summarize/labels/preview/sidecar |
| Q3 | Tipo de mock | Genérico/Semi-realista/Configurable |
| Q4 | Mock data pre-configurado | Sí incluir/No/Consumidor provee |
| Q5 | Paso de mock data | Config file/Direct API/Discovery |
| Q6 | Nivel de API | Basic/Detailed/Debug |
| Q7 | Fallback behavior | Fail fast/Graceful/Configurable |
| Q8 | Versioning | Semver/API version/Capabilities |
| Q9 | Versión mínima soportada | ¿? |
| Q10 | Instalación | PyPI/Local/Git submodule/Ambos |

---

## Recomendación Inicial

Basado en el contexto de "desarrollo en paralelo":

| Aspecto | Recomendación |
|---------|---------------|
| Detección | **Auto-detect** — más flexible |
| Funciones mínimas | **extract + preview** mínimo |
| Mock | **Semi-realista** — útil para testing real |
| Mock data | **Incluir fixtures** — facilita onboarding |
| Paso de mock | **Discovery automático** — menos configuración |
| Nivel API | **Detailed** — consumidor necesita saber layers usadas |
| Fallback | **Graceful** — no rompe consumidor si falta algo |
| Versioning | **Capabilities** — más flexible |
| Instalación | **Ambos** — PyPI + local editable para dev |
