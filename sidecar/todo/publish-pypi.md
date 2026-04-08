# Publicar en PyPI

Guía para publicar `sidecar_tagger` en PyPI usando `uv`.

## Prerrequisitos

1. Tener una cuenta en [PyPI](https://pypi.org/account/register/)
2. Tener `uv` instalado (`curl -LsSf https://astral.sh/uv/install.sh | sh`)
3. Tener un token de API de PyPI (recomendado) o credenciales de usuario

## Crear token de API en PyPI

1. Ir a [pypi.org/manage/account/token/](https://pypi.org/manage/account/token/)
2. Click en "Add API token"
3. Ponerle un nombre descriptivo (ej: `sidecar-tagger-ci`)
4. Copiar el token — empieza con `pypi-`

## Publicar por primera vez

```bash
# 1. Asegurarse de estar en la versión correcta
cat sidecar_tagger/__init__.py
# Debería decir: __version__ = "0.1.0"

# 2. Build del paquete
uv build

# 3. Verificar qué se va a subir
ls dist/
# Debería mostrar:
#   sidecar_tagger-0.1.0-py3-none-any.whl
#   sidecar_tagger-0.1.0.tar.gz

# 4. Subir a PyPI
uv publish --token pypi-TU_TOKEN_AQUI
```

## Publicar una nueva versión

```bash
# 1. Actualizar la versión en sidecar_tagger/__init__.py
#    Cambiar: __version__ = "0.1.0" → __version__ = "0.2.0"

# 2. Crear un tag de git (opcional pero recomendado)
git tag v0.2.0
git push origin v0.2.0

# 3. Build + publish
uv build
uv publish --token pypi-TU_TOKEN_AQUI
```

## Probar antes de publicar (TestPyPI)

PyPI tiene un entorno de pruebas donde podés subir sin afectar el paquete real:

```bash
# Subir a TestPyPI
uv publish --publish-url https://test.pypi.org/legacy/ --token pypi-TU_TOKEN_AQUI

# Instalar desde TestPyPI para probar
pip install --index-url https://test.pypi.org/simple/ sidecar_tagger
```

## Configurar token para no tener que pasarlo cada vez

```bash
# Guardar el token en UV_PUBLISH_TOKEN (una sola vez)
export UV_PUBLISH_TOKEN="pypi-TU_TOKEN_AQUI"

# O en .env del proyecto
echo 'UV_PUBLISH_TOKEN=pypi-TU_TOKEN_AQUI' >> .env

# Después solo:
uv build && uv publish
```

## Verificar que se publicó correctamente

```bash
# Buscar en PyPI
pip index versions sidecar_tagger

# Instalar y probar
pip install sidecar_tagger
sidecar-tag --help
```

## Versionado

El versionado se maneja en `sidecar_tagger/__init__.py`:

```python
__version__ = "0.1.0"  # Cambiar esto en cada release
```

Seguir [Semantic Versioning](https://semver.org/):

- `0.1.0` → `0.2.0` : nuevas features (minor)
- `0.1.0` → `0.1.1` : bug fixes (patch)
- `0.1.0` → `1.0.0` : primera versión estable (major)

## Troubleshooting

### "File already exists"

Significa que esa versión ya fue subida. PyPI no permite re-subir la misma versión.
Solución: incrementar la versión en `__init__.py` y hacer build de nuevo.

### "The user '...' isn't allowed to upload to project 'sidecar_tagger'"

Necesitás que el owner del proyecto te agregue como maintainer en PyPI,
o usar un token con permisos de upload.

### "No such file or directory: dist/"

Correr `uv build` primero para generar los archivos en `dist/`.
