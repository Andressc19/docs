# Issue 2: Migrar Storage a Ubicación Centralizada

**Estado:** Pendiente  
**Prioridad:** Alta  
**Estimación:** 2-3 días

---

## Descripción

Centralizar todos los datos en `~/.sidecar-tagger/` en lugar de generar `sidecar.json` en cada carpeta, evitando "mugre" en las carpetas del usuario.

---

## Problema Actual

 Actualmente el sistema genera `sidecar.json` en cada carpeta procesada:
 
 ```
 Documents/
 ├── Report.pdf
 ├── sidecar.json  ← "mugre"
 └── Invoices/
     ├── invoice1.pdf
     └── sidecar.json  ← "mugre"
 ```

**Issues:**
- Archivos innecesarios en cada carpeta
- No sobrevive cuando el usuario mueve archivos
- Difícil de mantener/migrar

---

## Estructura Propuesta

```
~/.sidecar-tagger/
├── config.json              # API keys, paths rastreados
├── manifest.json            # Index global (path → metadata summary)
├── metadata/                # Metadata detallada por hash
│   ├── ab/
│   │   └── abc123def456.json
│   ├── cd/
│   │   └── ...
│   └── ...
└── cache/
    └── embeddings/          # Embeddings (puede crecer mucho)
```

---

### manifest.json

```json
{
  "version": "2.0",
  "last_updated": "2026-03-27T12:00:00Z",
  "index": {
    "/home/user/Documents/report.pdf": {
      "file_hash": "abc123...",
      "doc_type": "invoice",
      "domain": "finance",
      "tags": ["azure", "billing"],
      "last_indexed": "2026-03-27T12:00:00Z"
    }
  }
}
```

---

### metadata/{hash}.json

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

---

## Features a Implementar

### 1. Storage Manager

```python
class CentralizedStorage:
    def __init__(self, base_path: str = "~/.sidecar-tagger"):
        self.base_path = Path(base_path).expanduser()
        self.manifest_path = self.base_path / "manifest.json"
        self.metadata_dir = self.base_path / "metadata"
    
    def get_metadata_path(self, file_hash: str) -> Path:
        """Retorna path para metadata por hash (sharding por prefijo)."""
        prefix = file_hash[:2]
        return self.metadata_dir / prefix / f"{file_hash}.json"
    
    def save_metadata(self, file_path: str, metadata: FileMetadata):
        """Guarda metadata en storage centralizado."""
        file_hash = metadata.file_hash
        meta_path = self.get_metadata_path(file_hash)
        meta_path.parent.mkdir(parents=True, exist_ok=True)
        # ... save JSON
    
    def get_metadata(self, file_path: str) -> Optional[FileMetadata]:
        """Recupera metadata por path desde manifest."""
        # ... load from manifest
```

---

### 2. Comando --reindex

```bash
# Re-index paths específicos
sidecar-tagger --reindex --scan-paths /home/user/Documents

# Re-index completo
sidecar-tagger --reindex --full
```

**Lógica de reindex:**
```
1. Cargar manifest existente
2. Para cada path en manifest:
   a. Verificar si archivo existe
   b. Si no existe:
      - Buscar por hash en filesystem
      - Si se encuentra → actualizar path
      - Si no → marcar como "orphan"
3. Limpiar orphans
4. Guardar manifest actualizado
```

---

### 3. Migración de sidecar.json existentes

```bash
# Importar sidecar.json legacy
sidecar-tagger --migrate /path/to/old/sidecar.json
```

**Lógica:**
- Leer cada entrada de sidecar.json legacy
- Importar a manifest + metadata/
- Opcional: eliminar old sidecar.json

---

### 4. CLI Commands

```python
# cli/commands.py
import click

@click.group()
def cli():
    pass

@cli.command()
@click.option('--scan-paths', multiple=True, help='Paths a re-indexar')
def reindex(scan_paths):
    """Re-index archivos, detectandomovidos/eliminados."""
    pass

@cli.command()
@click.argument('sidecar_file')
def migrate(sidecar_file):
    """Migrar sidecar.json legacy a storage centralizado."""
    pass
```

---

## Consideraciones

### Sharding por Hash

Usar prefijo del hash (2 chars) para evitar muchos archivos en un directorio:
- `metadata/ab/abc123...json`
- `metadata/cd/def456...json`

---

### Paths Relativos vs Absolutos

- Guardar siempre paths absolutos en manifest
- Si el usuario cambia de máquina → reindex necesario

---

### Backward Compatibility

- Mantener soporte para `sidecar.json` por carpeta (legacy)
- Flag `--use-centralized` para habilitar nuevo storage

---

## Tests Recomendados

- `test_storage_initialization()` - Crear estructura correcta
- `test_save_and_retrieve_metadata()` - Save/load cycle
- `test_reindex_detects_moved_files()` - Simular move
- `test_reindex_detects_deleted_files()` - Simular delete
- `test_migrate_legacy_sidecar()` - Migración correcta

---

## Métricas de Éxito

| Métrica | Target |
|---------|--------|
| Archivos por carpeta | 0 (sin "mugre") |
| Survive file moves | 100% (con reindex) |
| Tiempo de lookup | <10ms |
