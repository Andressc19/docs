# Issue 5: Sistema de Métricas y Reporter

**Estado:** Pendiente  
**Prioridad:** Media  
**Estimación:** 1-2 días

---

## Descripción

Agregar tracking de qué layers resuelven cada archivo durante el procesamiento y generar un reporte post-proceso (`findings.md`) con métricas y insights.

---

## Métricas a Tracking

### Por Layer

| Layer | Métrica | Descripción |
|-------|---------|-------------|
| Layer 0 | `hash_dedup_count` | Archivos con hash duplicado |
| Layer 0.5 | `native_metadata_count` | Archivos resueltos por metadata nativa |
| Layer 2 | `cluster_hint_count` | Archivos con hints de clustering |
| Layer 3 | `semantic_cache_count` | Archivos con cache semántico (embeddings) |
| Layer 4 | `llm_calls_count` | Archivos que requirieron LLM |

### Globales

| Métrica | Descripción |
|---------|-------------|
| `total_files` | Total de archivos procesados |
| `total_time` | Tiempo total de procesamiento |
| `total_cost` | Costo estimado de API |
| `errors_count` | Archivos con error |

---

## Estructura de Tracking

```python
# sdk/reporter/metrics.py
from dataclasses import dataclass, field
from typing import Dict
import time

@dataclass
class ProcessingMetrics:
    # Layer counts
    hash_dedup_count: int = 0
    native_metadata_count: int = 0
    cluster_hint_count: int = 0
    semantic_cache_count: int = 0
    llm_calls_count: int = 0
    
    # Globals
    total_files: int = 0
    total_time: float = 0.0
    total_cost: float = 0.0
    errors_count: int = 0
    
    # Details
    layer_details: Dict[str, list] = field(default_factory=dict)
    
    def record_layer(self, layer_name: str, file_path: str, details: dict = None):
        """Registrar qué layer resolvió un archivo."""
        attr = f"{layer_name}_count"
        if hasattr(self, attr):
            setattr(self, attr, getattr(self, attr) + 1)
            
            if layer_name not in self.layer_details:
                self.layer_details[layer_name] = []
            self.layer_details[layer_name].append({
                "file": file_path,
                "details": details or {}
            })
    
    def get_summary(self) -> dict:
        """Resumen de métricas."""
        return {
            "total_files": self.total_files,
            "layers": {
                "layer_0_hash": {
                    "count": self.hash_dedup_count,
                    "percentage": self.hash_dedup_count / self.total_files * 100 if self.total_files else 0,
                    "cost": 0
                },
                "layer_0_5_native": {
                    "count": self.native_metadata_count,
                    "percentage": self.native_metadata_count / self.total_files * 100 if self.total_files else 0,
                    "cost": 0
                },
                "layer_3_semantic": {
                    "count": self.semantic_cache_count,
                    "percentage": self.semantic_cache_count / self.total_files * 100 if self.total_files else 0,
                    "cost": 0
                },
                "layer_4_llm": {
                    "count": self.llm_calls_count,
                    "percentage": self.llm_calls_count / self.total_files * 100 if self.total_files else 0,
                    "cost": self.llm_calls_count * 0.01  # ~$0.01 per call
                }
            },
            "total_cost": self.total_cost,
            "total_time": self.total_time,
            "errors": self.errors_count
        }
```

---

## Reporter: findings.md

```python
# sdk/reporter/generator.py
from pathlib import Path

class FindingsReporter:
    def __init__(self, output_dir: str = "."):
        self.output_dir = Path(output_dir)
    
    def generate(self, metrics: ProcessingMetrics, files_data: list) -> str:
        """Genera reporte findings.md."""
        
        summary = metrics.get_summary()
        
        report = []
        report.append("# Findings Report")
        report.append(f"\nGenerated: {datetime.now().isoformat()}")
        report.append(f"\n## Summary")
        report.append(f"- **Total Files:** {summary['total_files']}")
        report.append(f"- **Total Time:** {summary['total_time']:.2f}s")
        report.append(f"- **Total Cost:** ${summary['total_cost']:.4f}")
        report.append(f"- **Errors:** {summary['errors']}")
        
        report.append(f"\n## Layer Resolution")
        for layer, data in summary['layers'].items():
            report.append(f"- **{layer}:** {data['count']} ({data['percentage']:.1f}%) - ${data['cost']:.4f}")
        
        # Duplicate savings
        report.append(f"\n## Duplicate Savings")
        report.append(f"Layer 0 (Hash) avoided {metrics.hash_dedup_count} LLM calls")
        report.append(f"Layer 3 (Semantic) avoided {metrics.semantic_cache_count} LLM calls")
        
        # Cost breakdown
        llm_cost = metrics.llm_calls_count * 0.01
        savings = (metrics.hash_dedup_count + metrics.semantic_cache_count) * 0.01
        report.append(f"\n## Cost Analysis")
        report.append(f"- **LLM Cost:** ${llm_cost:.4f}")
        report.append(f"- **Savings:** ${savings:.4f}")
        report.append(f"- **Net Cost:** ${llm_cost - savings:.4f}")
        
        # Cluster insights
        report.append(f"\n## Cluster Insights")
        # ... detectar patrones de carpetas
        
        # Anomalies
        report.append(f"\n## Anomalies")
        # ... archivos que no encajan en clusters
        
        return "\n".join(report)
    
    def save(self, content: str):
        """Guarda reporte a findings.md."""
        output_path = self.output_dir / "findings.md"
        output_path.write_text(content)
```

---

## Ejemplo de findings.md

```markdown
# Findings Report

Generated: 2026-03-27T12:00:00Z

## Summary
- **Total Files:** 100
- **Total Time:** 45.2s
- **Total Cost:** $0.15
- **Errors:** 2

## Layer Resolution
- **layer_0_hash:** 15 (15.0%) - $0.00
- **layer_0_5_native:** 20 (20.0%) - $0.00
- **layer_3_semantic:** 10 (10.0%) - $0.00
- **layer_4_llm:** 55 (55.0%) - $0.55

## Duplicate Savings
Layer 0 (Hash) avoided 15 LLM calls
Layer 3 (Semantic) avoided 10 LLM calls

## Cost Analysis
- **LLM Cost:** $0.55
- **Savings:** $0.25
- **Net Cost:** $0.30

## Cluster Insights
- `/Documents/Invoices` → 85% related to 'finance/invoices'
- `/Projects/2024/` → mixed content (analyzed separately)

## Anomalies
- `party.jpg` found in `/Contracts` (semantic outlier)
- `readme.txt` in `/Backup` (potential duplicate)
```

---

## Integración con Processor

```python
# sdk/processor.py
class MetadataProcessor:
    def __init__(self, ...):
        # ...
        self.metrics = ProcessingMetrics()
        self.start_time = None
    
    def process_files(self, file_paths: List[str]):
        self.start_time = time.time()
        
        for path in normalized_paths:
            # ... existing logic
            # After processing:
            layer_used = self._detect_layer_used(result)
            self.metrics.record_layer(layer_used, path)
        
        # Finalize
        self.metrics.total_time = time.time() - self.start_time
        self.metrics.total_files = len(file_paths)
        
        # Generate report
        reporter = FindingsReporter()
        report = reporter.generate(self.metrics, self.metadata_store)
        reporter.save(report)
```

---

## CLI Options

```bash
# CLI
--generate-report           # Generar findings.md
--report-dir ./output       # Directorio para reporte
--quiet                     # Sin output detallado
--verbose                   # Output detallado
```

---

## Tests Recomendados

- `test_metrics_tracking()` - Tracking correcto por layer
- `test_metrics_summary()` - Cálculos correctos
- `test_findings_generation()` - Reporte generado correctamente
- `test_cost_calculation()` - Costo estimado correcto
