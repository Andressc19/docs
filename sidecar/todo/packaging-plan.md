# Packaging Implementation Plan

## Cambios Completados

| # | Cambio | Commit | Estado |
|---|--------|--------|--------|
| 1 | `setup-pyproject-toml` | `5888abd` | ✅ Completado |
| 2 | `refactor-namespace` | `362e463` | ✅ Completado |
| 3 | `exiftool-runtime-check` | `32c845d` | ✅ Completado |
| 4 | `cleanup-packaging` | `81122f4` | ✅ Completado |

---

## Cambio 5: `rename-package` (Pendiente)

**Estado**: Bloqueado — nombre final del paquete no definido.

### Contexto

El nombre actual es `sidecar_tagger` (tanto en PyPI como en el namespace del código). Este nombre es temporal hasta definir el nombre final del producto.

**Por qué no se puede hacer ahora**: PyPI no permite renombrar paquetes. Una vez publicado `sidecar_tagger`, ese nombre queda registrado permanentemente. Si se cambia después, los usuarios actuales no reciben actualizaciones automáticamente y hay que mantener dos paquetes en paralelo.

### Qué se necesita decidir antes de ejecutar

1. **Nombre final del paquete** (ej: `sidecar`, `stag`, `file-tagger`, etc.)
2. **Si `sidecar_tagger` ya fue publicado en PyPI**:
   - Si NO: simplemente cambiar el nombre antes del primer publish
   - Si SI: hay que planear una migración (ver "Estrategia de migración" más abajo)

### Alcance (si NO se publicó nada aún)

Si el paquete nunca se publicó en PyPI, el cambio es simple:

1. Cambiar `name` en `pyproject.toml`: `sidecar_tagger` → `nuevo_nombre`
2. Renombrar carpeta `sidecar_tagger/` → `nuevo_nombre/`
3. Actualizar todos los imports: `from sidecar_tagger.` → `from nuevo_nombre.`
4. Actualizar todos los `@patch()` en tests
5. Actualizar `__version__` en `nuevo_nombre/__init__.py`
6. Actualizar entry point en `pyproject.toml`
7. Actualizar docs: README.md, docs/01-setup.md, docs/publish-pypi.md
8. Actualizar `.github/workflows/publish.yml` (si existe)

### Alcance (si YA se publicó en PyPI)

Si `sidecar_tagger` ya está publicado, la estrategia cambia:

**Opción A: Publicar nuevo nombre como paquete separado**
1. Publicar `nuevo_nombre` en PyPI como paquete independiente
2. En `sidecar_tagger`, agregar un warning de deprecación al startup
3. En el README de `sidecar_tagger`, indicar que se mudó a `nuevo_nombre`
4. Mantener `sidecar_tagger` con updates por un tiempo (ej: 6 meses)
5. Eventualmente hacer un último release de `sidecar_tagger` que solo instale `nuevo_nombre` como dependencia

**Opción B: Wrapper mínimo**
1. Publicar `sidecar_tagger` como un wrapper vacío que depende de `nuevo_nombre`
2. El usuario instala `sidecar_tagger` pero en realidad usa `nuevo_nombre`
3. Transparente para el usuario, pero hay que mantener dos paquetes

### Archivos afectados

| Archivo | Acción |
|---------|--------|
| `pyproject.toml` | Modificar `name` y entry point |
| `{sidecar_tagger,nuevo_nombre}/` | Renombrar carpeta |
| Todos los `.py` con imports | Actualizar paths |
| Todos los tests con `@patch()` | Actualizar strings |
| `README.md` | Actualizar nombre y ejemplos |
| `docs/01-setup.md` | Actualizar nombre y ejemplos |
| `docs/publish-pypi.md` | Actualizar nombre |
| `docs/security.md` | Actualizar nombre |
| `.github/workflows/publish.yml` | Actualizar nombre de proyecto |

### Estimación

- Si NO se publicó: ~30 min (mismo esfuerzo que el cambio 2: refactor-namespace)
- Si SI se publicó: ~2-4 horas (depende de la estrategia de migración elegida)

### Dependencias

- Ninguna — se puede hacer en cualquier momento después del cambio 2
- Pero es **mucho más barato** hacerlo antes del primer publish a PyPI

### Criterios de salida

- [ ] Nombre final definido y documentado
- [ ] Todos los imports actualizados
- [ ] Todos los tests pasan (72+)
- [ ] `uv build` funciona con el nuevo nombre
- [ ] Docs actualizados
- [ ] Si ya estaba publicado: estrategia de migración ejecutada
