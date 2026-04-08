# SPEC: Taxonomía de Labels

**Documento:** Sistema de etiquetas semánticas para clasificación de archivos  
**Referencia:** INIT_SPEC.md sección 6

---

## Enfoque: Híbrido

| Aspecto | Definición |
|---------|------------|
| **Base predefinida** | Lista fija de labels para MVP |
| **Configurable** | Consumidor puede extender via config |

---

## Labels Predefinidos (MVP)

```python
DEFAULT_LABELS = [
    # Categorías de Negocio/Legal
    "contract",      # Contratos, acuerdos, convenios
    "legal",         # Documentos legales, cláusulas
    "invoice",       # Facturas, recibos, pagos
    "finance",       # Documentos financieros en general
    
    # Proveedores y externos
    "provider",      # Documentos de proveedores, vendor docs
    
    # Documentación técnica
    "architecture",  # Diagramas, specs técnicas, RFCs
    "spec",          # Especificaciones formales
    "config",        # Configuraciones, settings
    
    # Estados de documento
    "draft",         # Borradores, versiones preliminares
    "notes",         # Notas, comentarios
    "reference",     # Material de referencia
    
    # Utilidades
    "temp",          # Temporales, basura
]
```

---

## Grupos Semánticos

```
Categorías de Negocio/Legal
├── contract
├── legal
├── invoice
└── finance

Proveedores/Externos
└── provider

Documentación Técnica
├── architecture
├── spec
└── config

Estados de Documento
├── draft
├── notes
└── reference

Utilidades
└── temp
```

---

## Ejemplo de Uso

### Sugerencia de Labels

```python
# Input: PDF de contrato con proveedor
labels = suggest_labels("contrato-proveedor-2026.pdf")
# Output:
{
    "suggested_labels": ["contract", "provider"],
    "confidence": {
        "contract": 0.92,
        "provider": 0.85,
        "finance": 0.30
    },
    "reasoning": "Document titled 'Acuerdo de Proveedores' contains contract terms..."
}
```

### Configuración del Consumidor

```yaml
# config.yaml del consumidor
filekor:
  labels:
    taxonomy: custom
    custom_labels:
      - urgente
      - para-revisar
      - aprobado
      - rechazado
    confidence_threshold: 0.7
```

---

## Estructura en Sidecar

```json
{
  "labels": {
    "suggested": ["contract", "provider"],
    "confidence": {
      "contract": 0.92,
      "provider": 0.85,
      "finance": 0.30
    },
    "user_override": null,
    "final": ["contract", "provider"]
  }
}
```

---

## Algoritmo de Sugerencia

### Layer 1 (Fast)

Basado en keywords del filename y path:

```python
def suggest_from_path(file_path: Path) -> List[str]:
    path_str = str(file_path).lower()
    keywords = {
        "contract": ["contract", "contrato", "agreement", "acuerdo"],
        "invoice": ["invoice", "factura", "receipt", "recibo"],
        "finance": ["finance", "financiero", "budget", "presupuesto"],
        # ...
    }
```

### Layer 3 (Deep)

Analisis semántico con LLM:

```python
def suggest_from_content(text: str, taxonomy: List[str]) -> LabelSuggestion:
    prompt = f"""
    Analyze this document and suggest labels from the taxonomy.
    
    Taxonomy: {taxonomy}
    
    Document: {text[:2000]}  # primeros 2000 chars
    
    Return JSON with:
    - suggested_labels: list
    - confidence: dict
    - reasoning: str
    """
```

---

## Confidence Scoring

| Score Range | Significado | Acción |
|------------|-------------|--------|
| 0.9 - 1.0 | Muy seguro | Auto-assign |
| 0.7 - 0.9 | Seguro | Sugerir strongly |
| 0.5 - 0.7 | Posible | Sugerir weakly |
| 0.0 - 0.5 | Improbable | No sugerir |

---

## Extensibilidad

El consumidor puede:

1. **Agregar labels custom** via config
2. **Modificar thresholds** de confianza
3. **Ignorar labels predefinidos** que no use
4. **Crear grupos jerárquicos** propios

```python
# Ejemplo de config extendida
{
  "labels": {
    "use_defaults": true,
    "custom_labels": ["urgente", "reviewed"],
    "custom_groups": {
      "workflow": ["urgente", "reviewed", "archived"]
    }
  }
}
```
