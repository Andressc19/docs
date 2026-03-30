# Evolución de la Arquitectura de Layers

> Resumen de cambios realizados en la definición de capas del pipeline
> Fecha: 2026-03-30

---

## Estructura Anterior (Original)

| Layer | Nombre | Descripción |
|-------|--------|-------------|
| Layer 0 | Hash Dedup | SHA256 para duplicados |
| Layer 0.5 | Native Metadata | Metadata nativa (EXIF, ID3, PDF props) |
| Layer 1 | OS Context | Filename, path, timestamps |
| Layer 2 | Clustering | Agrupar por similaridad de nombre |
| Layer 3 | Embeddings | FastEmbed vectorización |
| Layer 4 | LLM | Gemini para metadata |

**Problemas identificados:**
1. Layer 0.5 y Layer 1 eran redundantes (solo ~5 campos de diferencia)
2. Layer 2 (Clustering) podía taggear mal archivos si solo usaba nombre
3. Demasiadas capas para un MVP

---

## Nueva Estructura (MVP)

| Layer | Nombre | Descripción |
|-------|--------|-------------|
| **Layer 0** | Hash Dedup | SHA256 - si existe, return cached |
| **Layer 1** | Native + OS | EXIFTOOL + filename, path, size, timestamps |
| **Layer 2** | Embeddings | FastEmbed - similarity check |
| **Layer 3** | LLM + Hint | Gemini + Clustering Hint como contexto |

### Cambios Clave

#### 1. Layer 0.5 + Layer 1 → Layer 1 (Unificado)

**Antes:**
```
Layer 0.5: author, title, keywords (DEL ARCHIVO)
Layer 1:   filename, path, timestamps (DEL SISTEMA)
```

**Después:**
```
Layer 1: Combina ambos (Native + OS)
```

**Rationale:** Solo son ~7 campos extra del OS. No justifica una capa separada.

#### 2. Layer 2 (Clustering) → Hint para LLM

**Antes:** Layer 2 era una capa que heredaba metadata automáticamente

**Después:** Clustering ahora es un **hint contextual** que se pasa al LLM en Layer 3

**Rationale:**
- El clustering por nombre puede taggear mal un archivo nuevo
- Ejemplo: `invoice_2024.pdf` y `my_invoice.txt` parecerían similares por nombre
- Mejor usar como contexto, no como verdad absoluta

#### 3. Layers reducidos: 5 → 4

**Antes:** Layer 0, 0.5, 1, 2, 3, 4 (6 layers)

**Después:** Layer 0, 1, 2, 3 (4 layers)

---

## Flujo del Pipeline

```
┌─────────────────────────────────────────────────────────────────┐
│  Layer 0: Hash Dedup                                            │
│  → Si existe en cache → return metadata (costo: $0)           │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  Layer 1: Native + OS                                           │
│  Native: EXIFTOOL (author, title, keywords, EXIF)             │
│  OS:     filename, path, size, timestamps                      │
│  → Si confidence ≥ 0.8 → return metadata (costo: $0)          │
│  → Si confidence < 0.8 → continuar                             │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  Layer 2: Embeddings                                            │
│  → Genera embedding del contenido                              │
│  → Si similarity ≥ 0.9 con otro archivo → return cached       │
│  → Si no → continuar                                            │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  Layer 3: LLM + Clustering Hint                                 │
│  Prompt incluye:                                                │
│    - Filename + path                                            │
│    - Native metadata                                            │
│    - Clustering hint (contexto de carpeta)                     │
│  → Gemini genera metadata final (costo: $)                      │
└─────────────────────────────────────────────────────────────────┘
```

---

## Niveles de Análisis

| Nivel | Capas | Costo | Shortcut |
|-------|-------|-------|----------|
| **minimal** | L0 | $0 | Hash |
| **fast** | L0 + L1 | $0 | Native+OS |
| **standard** | L0 + L1 + L2 | $0 | Embeddings |
| **deep** | L0 + L1 + L2 + L3 | $ | - |

---

## Comparativa

| Aspecto | Antes | Ahora |
|---------|-------|-------|
| Total layers | 6 | 4 |
| Clustering | Capa separada | Hint para LLM |
| Native + OS | 2 layers | 1 layer |
| Shortcuts | 2 (0.5, 3) | 3 (1, 2, 3) |
| Complejidad | Alta | Media |

---

## Decisiones Técnicas

1. **Usar EXIFTOOL** - No reinventar la wheel para metadata nativa
   - 25,000+ formatos soportados
   - Cross-platform (Windows, macOS, Linux)
   - CLI maduro y estable

2. **Confidence threshold = 0.8** - Balance entre calidad y cobertura
   - Requiere author + title = 0.5
   - keywords/subject = +0.2
   - date = +0.2

3. **Embedding similarity = 0.9** - Estricto para evitar falsos positivos

---

## Issues del MVP

| # | Issue | Descripción |
|---|-------|-------------|
| 1 | Analysis Levels + Tests | Niveles configurables + tests de integración |
| 2 | Layer 1 Native + OS | EXIFTOOL + metadata del sistema |

---

## Referencias

- [MVP.md](../mvp/MVP.md)
- [file-extensions.md](../mvp/file-extensions.md)
- [check/01-analysis-levels.md](../check/01-analysis-levels.md)
- [check/02-exiftool-native-metadata.md](../check/02-exiftool-native-metadata.md)
