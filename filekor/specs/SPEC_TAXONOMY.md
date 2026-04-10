# SPEC: Label Taxonomy

**Document:** Semantic tagging system for file classification  
**Reference:** INIT_SPEC.md section 6

---

## Approach: Hybrid

| Aspect | Definition |
|--------|------------|
| **Predefined base** | Fixed list of labels for MVP |
| **Configurable** | Consumer can extend via config |

---

## Predefined Labels (MVP)

```python
DEFAULT_LABELS = [
    # Business/Legal Categories
    "contract",      # Contracts, agreements, conventions
    "legal",         # Legal documents, clauses
    "invoice",       # Invoices, receipts, payments
    "finance",       # Financial documents in general
    
    # Providers and External
    "provider",      # Provider documents, vendor docs
    
    # Technical Documentation
    "architecture",  # Diagrams, technical specs, RFCs
    "spec",          # Formal specifications
    "config",        # Configurations, settings
    
    # Document States
    "draft",         # Drafts, preliminary versions
    "notes",         # Notes, comments
    "reference",     # Reference material
    
    # Utilities
    "temp",          # Temporary, trash
]
```

---

## Semantic Groups

```
Business/Legal Categories
├── contract
├── legal
├── invoice
└── finance

Providers/External
└── provider

Technical Documentation
├── architecture
├── spec
└── config

Document States
├── draft
├── notes
└── reference

Utilities
└── temp
```

---

## Usage Example

### Label Suggestion

```python
# Input: PDF contract with provider
labels = suggest_labels("contrato-proveedor-2026.pdf")
# Output:
{
    "suggested_labels": ["contract", "provider"],
    "confidence": {
        "contract": 0.92,
        "provider": 0.85,
        "finance": 0.30
    },
    "reasoning": "Document titled 'Provider Agreement' contains contract terms..."
}
```

### Consumer Configuration

```yaml
# consumer config.yaml
filekor:
  labels:
    taxonomy: custom
    custom_labels:
      - urgent
      - to-review
      - approved
      - rejected
    confidence_threshold: 0.7
```

---

## Structure in Sidecar

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

## Suggestion Algorithm

### Layer 1 (Fast)

Based on filename and path keywords:

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

Semantic analysis with LLM:

```python
def suggest_from_content(text: str, taxonomy: List[str]) -> LabelSuggestion:
    prompt = f"""
    Analyze this document and suggest labels from the taxonomy.
    
    Taxonomy: {taxonomy}
    
    Document: {text[:2000]}  # first 2000 chars
    
    Return JSON with:
    - suggested_labels: list
    - confidence: dict
    - reasoning: str
    """
```

---

## Confidence Scoring

| Score Range | Meaning | Action |
|-------------|---------|--------|
| 0.9 - 1.0 | Very sure | Auto-assign |
| 0.7 - 0.9 | Sure | Strongly suggest |
| 0.5 - 0.7 | Possible | Weakly suggest |
| 0.0 - 0.5 | Improbable | Do not suggest |

---

## Extensibility

The consumer can:

1. **Add custom labels** via config
2. **Modify confidence thresholds**
3. **Ignore predefined labels** not used
4. **Create hierarchical groups** of their own

```python
# Example extended config
{
  "labels": {
    "use_defaults": true,
    "custom_labels": ["urgent", "reviewed"],
    "custom_groups": {
      "workflow": ["urgent", "reviewed", "archived"]
    }
  }
}
```