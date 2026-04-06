# Packaging Decisions for sidecar-tagger

Checklist para responder cada pregunta. Marca la opción elegida con [X].

## 1. Requirements.txt vs pyproject.toml
¿Querés mantener `requirements.txt` para desarrollo local o migrar todo a `pyproject.toml` como fuente de verdad?

[X] Opción A: Migrar todo a `pyproject.toml`, eliminar `requirements.txt`.
[ ] Opción B: Mantener ambos.

**Comentarios:** Revisar readme y documentación para que actualizada luego de este cambio

## 2. Package manager
¿Qué tooling preferís para build/publish?

[ ] Opción A: `pip` + `build` + `twine` (vanilla)
[X] Opción B: `uv`
[ ] Opción C: `poetry`

**Comentarios:** Dejar especificado en la documentación como usarlo y dependencias

## 3. Namespace del paquete
La estructura actual tiene `cli/` y `sdk/` como folders sueltos. Para un package Python necesitamos un namespace.

[X] Opción A: `sidecar_tagger/` (mover todo adentro)
[ ] Opción B: Mantener estructura actual con entry point directo

## 4. Package name en PyPI
¿Qué nombre registrar en PyPI?

[ ] Opción A: `sidecar-tagger` (con guion)
[X] Opción B: `sidecar_tagger` (con underscore)

## 5. Versioning strategy
¿Cómo manejar las versiones?

[ ] Opción A: Hardcodear en `pyproject.toml`
[X] Opción B: `__version__` en el código (single source)
[ ] Opción C: Git tags (automático con CI/CD)

## 6. Dependencies con binarios nativos
ExifTool es una dependencia externa del sistema (no es un pip package). ¿Cómo manejarlo?

[ ] Opción A: Documentar ExifTool como prerequisito en metadata
[ ] Opción B: Solo documentar en README
[X] Opción C: Runtime check + warning

## 7. Python version support
¿Cuál es la versión mínima de Python que soportas?

[ ] Opción A: Python 3.11+ (más amplio)
[X] Opción B: Python 3.12+ (más moderno)
[ ] Opción C: Python 3.13+ (bleeding edge)

## Resumen de decisiones elegidas

1. [X] Migrar todo a `pyproject.toml`
2. [X] Usar `uv`
3. [X] Namespace: `sidecar_tagger/`
4. [X] Package name: `sidecar_tagger` (underscore)
5. [X] Versioning: `__version__` en código
6. [X] ExifTool: Runtime check + warning
7. [X] Python version: 3.12+

## Próximos pasos

Una vez confirmadas estas decisiones, el siguiente paso es crear un plan SDD para implementar los cambios:
- Crear `pyproject.toml`
- Mover código a `sidecar_tagger/`
- Actualizar imports
- Crear `__init__.py` con `__version__`
- Agregar runtime check para ExifTool
- Actualizar tests
- Eliminar `requirements.txt`
- Documentar en docs/