# Seguridad en Publicación de PyPI

Guía de seguridad para publicar y mantener el paquete `sidecar_tagger`.

---

## 1. Trusted Publishing (Recomendado)

En vez de usar tokens de API de PyPI (secretos de larga duración), PyPI soporta **Trusted Publishing** con OIDC de GitHub.

### Por qué es más seguro

- No hay tokens que guardar ni rotar
- Expira automáticamente después de cada publish
- Está scoping a un repo y branch específico
- Si te hackean la cuenta de GitHub, no pueden publicar en PyPI sin pasar por GitHub Actions

### Configuración en PyPI

1. Ir a [pypi.org/manage/account/publishing/](https://pypi.org/manage/account/publishing/)
2. Click en "Register a pending publisher"
3. Completar:
   - **Project name**: `sidecar_tagger`
   - **Owner**: `latai-community`
   - **Repository**: `sidecar-tagger`
   - **Workflow**: `.github/workflows/publish.yml`
   - **Environment**: `release` (opcional)
   - **Branch**: `main`

### GitHub Action

Crear `.github/workflows/publish.yml`:

```yaml
name: Publish to PyPI

on:
  release:
    types: [published]

permissions:
  id-token: write  # Required for OIDC
  contents: read

jobs:
  publish:
    runs-on: ubuntu-latest
    environment: release  # Si configuraste environment en PyPI
    steps:
      - uses: actions/checkout@v4

      - uses: astral-sh/setup-uv@v5

      - name: Build package
        run: uv build

      - name: Publish to PyPI
        run: uv publish
        # No necesita token! PyPI confía en GitHub OIDC
```

### Cómo funciona el flujo

1. Creás un release en GitHub (`v0.2.0`)
2. GitHub Actions se dispara automáticamente
3. `uv build` genera el `.whl` y `.tar.gz`
4. `uv publish` usa el token OIDC temporal de GitHub
5. PyPI verifica la identidad y publica

---

## 2. Tokens de API (Alternativa)

Si no querés configurar Trusted Publishing, usá tokens de API.

### Crear el token

1. Ir a [pypi.org/manage/account/token/](https://pypi.org/manage/account/token/)
2. Click en "Add API token"
3. Nombre: `sidecar-tagger-ci`
4. Scope: `Entire account` o solo `sidecar_tagger`
5. Copiar el token (empieza con `pypi-`)

### Guardar en GitHub Secrets

1. Ir a `Settings > Secrets and variables > Actions`
2. Click en "New repository secret"
3. Nombre: `UV_PUBLISH_TOKEN`
4. Valor: `pypi-TU_TOKEN_AQUI`

### GitHub Action con token

```yaml
name: Publish to PyPI

on:
  release:
    types: [published]

jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v5
      - run: uv build
      - run: uv publish
        env:
          UV_PUBLISH_TOKEN: ${{ secrets.UV_PUBLISH_TOKEN }}
```

### Rotar el token

Cada 6-12 meses:
1. Crear nuevo token en PyPI
2. Actualizar el GitHub Secret
3. Eliminar el token viejo

---

## 3. Scanning de Dependencias

Auditar dependencias en busca de vulnerabilidades conocidas.

### Herramienta: pip-audit

```bash
# Instalar
uv tool install pip-audit

# Escanear el proyecto
pip-audit
```

### Agregar a CI

```yaml
- name: Security audit
  run: |
    uv sync
    uv tool run pip-audit
```

### Automatizar con Dependabot

Crear `.github/dependabot.yml`:

```yaml
version: 2
updates:
  - package-ecosystem: "pip"
    directory: "/"
    schedule:
      interval: "weekly"
    open-pull-requests-limit: 10
```

Dependabot crea PRs automáticos cuando hay actualizaciones de seguridad.

---

## 4. Proteger el Repositorio

### Branch Protection (main)

1. Ir a `Settings > Branches > Add rule`
2. Pattern: `main`
3. Activar:
   - [x] Require a pull request before merging
   - [x] Require approvals (1 mínimo)
   - [x] Require status checks to pass before merging
   - [x] Require branches to be up to date before merging
   - [x] Include administrators (para que aplique a todos)

### Require Signed Commits (Opcional)

1. Ir a `Settings > Code security and analysis`
2. Activar "Require signed commits"
3. Configurar GPG en tu máquina local

---

## 5. Secretos Locales

### .gitignore

Verificar que estos archivos estén en `.gitignore`:

```
.env
.env.local
.env.*.local
*.key
*.pem
```

### .env.example

Mantener un `.env.example` con las variables pero sin valores reales:

```
GEMINI_API_KEY=
EXIFTOOL_PATH=
```

### NUNCA hacer esto

```bash
# MAL - commitear secretos
git add .env
git commit -m "add api key"

# MAL - hardcodear en código
API_KEY = "sk-1234567890"

# MAL - publicar desde tu máquina local en producción
uv publish --token pypi-xxxx
```

---

## 6. Checklist de Seguridad Pre-Publicación

Antes de cada release:

- [ ] No hay secretos commiteados (`git log -p | grep -i "key\|token\|secret\|password"`)
- [ ] `.env` está en `.gitignore`
- [ ] Dependencias actualizadas (`uv sync` sin warnings)
- [ ] `pip-audit` no reporta vulnerabilidades críticas
- [ ] Tests pasan (`uv run pytest`)
- [ ] Build funciona (`uv build` sin errores)
- [ ] Versión actualizada en `sidecar_tagger/__init__.py`
- [ ] CHANGELOG.md actualizado

---

## 7. Respuesta a Incidentes

### Si se filtra un token de PyPI

1. Ir a [pypi.org/manage/account/token/](https://pypi.org/manage/account/token/)
2. Revocar el token comprometido inmediatamente
3. Crear uno nuevo
4. Actualizar GitHub Secrets
5. Revisar logs de PyPI para ver si se usó maliciosamente

### Si se publica una versión con un bug crítico

```bash
# PyPI no permite borrar versiones (solo yank)
# Hacer "yank" de la versión problemática:
twine yank sidecar_tagger-0.2.0

# Publicar un fix con nueva versión
# __version__ = "0.2.1"
uv build && uv publish
```

### Si te hackean la cuenta de GitHub

1. Recuperar acceso a GitHub
2. Revocar todos los tokens de PyPI
3. Rotar todas las credenciales
4. Revisar activity log de PyPI
