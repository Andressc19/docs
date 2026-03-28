# Issue 6: Análisis Diferido (Async Queue)

**Estado:** Pendiente  
**Prioridad:** Baja  
**Estimación:** 3-4 días

---

## Descripción

Implementar procesamiento en background queue para no bloquear la espera de respuesta LLM. El usuario puede usar metadata parcial mientras el procesamiento completo corre en background.

---

## Problema Actual

Procesamiento síncrono:
1. Usuario ejecuta comando
2. Espera 1-3 segundos por archivo
3. Recibe metadata completa

**Issue:** Si son 1000 archivos → horas de espera.

---

## Solución Propuesta

```
┌─────────────────────────────────────────────────────────┐
│                    User Space                           │
│                                                         │
│  $ sidecar-tagger scan /Documents                      │
│                                                         │
│  Output:                                                │
│  - 500 archivos Layer 0-3: instantáneo                │
│  - 500 archivos Layer 4: enqueue (pending)            │
│                                                         │
│  → Usuario puede trabajar, no espera                   │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│                 Background Queue                        │
│                                                         │
│  Queue: [file1, file2, ..., file500]                   │
│  │                                                     │
│  ├─── Worker 1 → LLM → save → notify                  │
│  ├─── Worker 2 → LLM → save → notify                  │
│  └─── Worker 3 → LLM → save → notify                  │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## Arquitectura

```python
# sdk/queue/processor.py
from queue import Queue
from threading import Thread
from dataclasses import dataclass
import asyncio

@dataclass
class QueuedFile:
    file_path: str
    priority: int = 0  # 0 = normal, 1 = high

class AsyncProcessor:
    def __init__(self, max_workers: int = 3):
        self.queue = Queue()
        self.max_workers = max_workers
        self.results = {}
    
    def enqueue(self, file_path: str, priority: int = 0):
        """Encola archivo para procesamiento async."""
        self.queue.put(QueuedFile(file_path, priority))
    
    def start_workers(self):
        """Inicia workers en background."""
        for i in range(self.max_workers):
            worker = Thread(target=self._worker, daemon=True)
            worker.start()
    
    def _worker(self):
        """Worker que procesa archivos de la cola."""
        while True:
            item = self.queue.get()
            try:
                result = self.processor.extract_metadata(item.file_path)
                self.results[item.file_path] = result
                
                # Guardar resultado
                self.storage.save(item.file_path, result)
                
                # Notificar (webhook, callback, etc.)
                self._notify_completion(item.file_path, result)
            finally:
                self.queue.task_done()
    
    def wait_completion(self, timeout: int = None):
        """Espera hasta que todos los archivos estén listos."""
        self.queue.join()
```

---

## Estados de Metadata

```python
# sdk/models/metadata.py
from enum import Enum

class ProcessingStatus(Enum):
    PENDING = "pending"
    PROCESSING = "processing"
    COMPLETED = "completed"
    FAILED = "failed"

@dataclass
class FileMetadata:
    file_path: str
    status: ProcessingStatus = ProcessingStatus.PENDING
    # ... rest of metadata
```

---

## Manifest con Estados

```json
{
  "index": {
    "/Documents/report.pdf": {
      "status": "completed",
      "doc_type": "invoice",
      "last_updated": "2026-03-27T12:00:00Z"
    },
    "/Documents/new.pdf": {
      "status": "pending",
      "doc_type": null,
      "last_updated": null
    }
  }
}
```

---

## CLI

```bash
# Procesar async con 3 workers
sidecar-tagger scan /Documents --async --workers 3

# Esperar completion
sidecar-tagger wait

# Ver status
sidecar-tagger status --watch

# Procesar solo los pending
sidecar-tagger process-pending
```

---

## Notificaciones

```python
# Opcional: Notificaciones cuando termina un archivo
def on_complete(file_path: str, result: FileMetadata):
    """Callback cuando un archivo termina."""
    # print(f"✓ {file_path} done")
    # slack.notify(...)
    # webhook.post(...)

# Configuración
sidecar-tagger scan /Docs --async --notify
```

---

## Trade-offs

| Aspecto | Síncrono | Async |
|---------|----------|-------|
| Metadata inmediata | ✅ | ⚠️ Parcial |
| Bloquea UI/CLI | ✅ | ❌ |
| Complejidad | Simple | Media |
| Recursos | Bajo | Medio |

---

## Tests Recomendados

- `test_async_enqueue_dequeue()` - Queue funciona
- `test_worker_processes_items()` - Workers procesan
- `test_pending_status_in_manifest()` - Status actualizado
- `test_wait_completion()` - Espera correcta

---

## Métricas Esperadas

| Métrica | Target |
|---------|--------|
| Tiempo hasta metadata parcial | <1s |
| Throughput (archivos/seg) | 3x (parallel LLM) |
