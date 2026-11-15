---
icon: lucide/play
---

# CI/CD con GitHub Actions

**GitHub Actions** ejecuta *workflows* automáticamente cuando ocurre algo en tu repositorio (un `push`, un *pull request*...). Aquí vemos cómo es uno, analizamos el que publica esta misma web y montamos uno propio con pruebas y publicación de una imagen Docker.

Fuentes: [práctica guiada de CI/CD con GitHub Actions](https://dpl-f1facc.gitlab.io/ut01/practicas/guiada-25-github-actions-cicd/) (curso DPL) y la [teoría de CI/CD](teoria.md).

## Anatomía de un workflow

Los *workflows* son archivos YAML en `.github/workflows/`. GitHub los detecta solo; no hay que registrarlos en ningún sitio.

```yaml
name: Mi workflow          # nombre visible en la pestaña Actions
on:                        # CUÁNDO se ejecuta
  push:
    branches: [main]
jobs:                      # QUÉ se ejecuta
  prueba:
    runs-on: ubuntu-latest # máquina virtual del trabajo
    steps:                 # pasos secuenciales
      - uses: actions/checkout@v4    # usa una acción ya hecha
      - run: echo "Hola"             # ejecuta un comando
```

| Elemento | Significado |
|---|---|
| `on` | Eventos que disparan el workflow (`push`, `pull_request`, `workflow_dispatch`...). |
| `jobs` | Trabajos. Cada uno corre en su propia máquina virtual y, por defecto, **en paralelo**. |
| `runs-on` | Sistema operativo del *runner* donde se ejecuta el trabajo. |
| `steps` | Pasos de un trabajo, en orden. Si uno falla, el trabajo se detiene. |
| `uses` | Ejecuta una **acción** reutilizable (por ejemplo, `actions/checkout`). |
| `run` | Ejecuta un comando de terminal. |
| `needs` | Hace que un trabajo espere a otro. |
| `if` | Condición para ejecutar un trabajo o un paso. |

## Un ejemplo real: el que publica esta web

Este repositorio tiene `.github/workflows/docs.yml`, que construye la documentación con Zensical y la publica en GitHub Pages cada vez que algo llega a `main`:

```yaml
name: Documentation
on:
  push:
    branches:
      - master
      - main
permissions:
  contents: read
  pages: write
  id-token: write
jobs:
  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    steps:
      - uses: actions/configure-pages@v6
      - uses: actions/checkout@v7
      - uses: actions/setup-python@v6
        with:
          python-version: 3.x
      - run: pip install zensical
      - run: zensical build --clean
      - uses: actions/upload-pages-artifact@v5
        with:
          path: site
      - uses: actions/deploy-pages@v5
        id: deployment
```

Línea a línea:

| Parte | Qué hace |
|---|---|
| `on.push.branches` | Se dispara solo con un `push` a `main` (o `master`). Los *pull requests* **no** lo ejecutan: la web solo se publica al fusionar. |
| `permissions` | Permisos mínimos: leer el código, escribir en Pages y pedir un token de identidad para desplegar. |
| `environment: github-pages` | Marca el trabajo como un despliegue y muestra la URL final en la interfaz. |
| `runs-on: ubuntu-latest` | Se ejecuta en una máquina virtual Ubuntu de GitHub. |
| `actions/configure-pages` | Prepara GitHub Pages. |
| `actions/checkout` | **Source:** descarga el repositorio en la máquina. |
| `actions/setup-python` | Instala Python. |
| `pip install zensical` | **Build:** instala el generador de la web. |
| `zensical build --clean` | Construye el sitio en la carpeta `site/`. |
| `actions/upload-pages-artifact` | **Package:** guarda `site/` como artefacto. |
| `actions/deploy-pages` | **Deploy:** publica el artefacto en Pages. |

Fíjate en que sigue las fases de un *pipeline* de la [teoría](teoria.md): source, build, package y deploy. No tiene fase de pruebas, porque una web estática no las necesita, pero sí te avisa si el *build* falla.

!!! tip "Pruébalo antes de subirlo"
    El paso de *build* es el mismo comando que usas en local. Si `uv run zensical build --clean` falla en tu máquina, también fallará en GitHub.

## Práctica: pruebas y publicación de una imagen

Vamos a montar un *workflow* que, en cada `push` y *pull request*, ejecute las pruebas de una pequeña aplicación Python. Solo en `main` construirá y publicará una imagen Docker en **GHCR**, el registro de contenedores de GitHub.

```text
push o PR → job test → ¿push a main? ── sí → job build-and-push → ghcr.io
                                      └─ no (PR) → fin
```

Necesitas Git, Docker, una cuenta de GitHub y Python. La GitHub CLI (`gh`) es opcional.

### Paso 1: la aplicación y sus pruebas

Crea una carpeta con tres archivos.

`app.py`:

```python
def suma(a, b):
    return a + b

def es_par(n):
    return n % 2 == 0
```

`test_app.py`:

```python
from app import suma, es_par

def test_suma():
    assert suma(2, 3) == 5

def test_es_par():
    assert es_par(4)
    assert not es_par(3)
```

`requirements.txt`:

```text
pytest==8.3.2
```

### Paso 2: el Dockerfile

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY app.py .
CMD ["python", "-c", "from app import suma; print('2 + 3 =', suma(2, 3))"]
```

### Paso 3: comprobarlo en local

Antes de automatizar nada, comprueba que funciona a mano:

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
pytest -q
docker build -t guiada-25:local .
docker run --rm guiada-25:local
```

Deberías ver `2 passed` y después `2 + 3 = 5`. Crea también un `.gitignore` para no versionar el entorno virtual:

```text
.venv/
__pycache__/
*.pyc
```

### Paso 4: subirlo a GitHub

Crea un repositorio (con `gh repo create` o desde la web), añade el remoto y haz `git push -u origin main`.

### Paso 5: el workflow

Crea `.github/workflows/ci-cd.yml`:

```yaml
name: CI/CD
on:
  push:
    branches: [main]
  pull_request:
  workflow_dispatch:

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - run: pip install -r requirements.txt
      - run: pytest -q

  build-and-push:
    needs: test
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
      - uses: actions/checkout@v4
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: ghcr.io/${{ github.repository }}:${{ github.sha }}
```

Qué hace cada pieza:

| Clave | Función |
|---|---|
| `on.pull_request` | Valida los *pull requests* sin publicar nada. |
| `on.workflow_dispatch` | Permite lanzarlo a mano desde la pestaña Actions. |
| `needs: test` | Si las pruebas fallan, **no se construye ni se publica** la imagen. |
| `if: github.ref == ... && github.event_name == 'push'` | Publica solo en un `push` a `main`, nunca desde un PR. |
| `permissions.packages: write` | El permiso mínimo para subir la imagen a GHCR. |
| `secrets.GITHUB_TOKEN` | Token que GitHub genera solo en cada ejecución. **No hay credenciales en el archivo.** |
| `tags: ...:${{ github.sha }}` | Etiqueta la imagen con el hash del commit, para saber de qué código sale. |

Haz commit y `git push origin main`.

### Paso 6: comprobar el resultado

En la pestaña **Actions** del repositorio deberías ver:

- El trabajo `test` en verde.
- El trabajo `build-and-push` ejecutado, porque estás en `main`.

Con la GitHub CLI:

```bash
gh run list --limit 5
gh run watch
```

En el apartado **Packages** del repositorio aparece la imagen, con el hash del commit como etiqueta.

Prueba ahora un *pull request*: crea una rama, cambia algo y ábrelo. Solo debe ejecutarse `test`; `build-and-push` no.

### Paso 7: probar la imagen publicada

```bash
docker pull ghcr.io/TU_USUARIO/TU_REPO:HASH_DEL_COMMIT
docker run --rm ghcr.io/TU_USUARIO/TU_REPO:HASH_DEL_COMMIT
```

Debe imprimir `2 + 3 = 5`. Si el paquete es privado, tendrás que hacer `docker login ghcr.io` con un token; nunca escribas ese token en el repositorio.

## Problemas frecuentes

| Problema | Causa probable | Solución |
|---|---|---|
| El workflow no aparece en Actions | El archivo no está en `.github/workflows/` o no se hizo `push` a la rama correcta | Revisa la ruta con `git ls-files .github` |
| `test` falla | Dependencias o pruebas rotas | Mira el registro con `gh run view --log` y reproduce `pytest -q` en local |
| Falla el inicio de sesión en `ghcr.io` | Falta el permiso de escritura de paquetes | Añade `packages: write` en `permissions` |
| `build-and-push` no se ejecuta | La condición `if` no se cumple | Comprueba que es un `push` a `main`, no un PR |
| La imagen no aparece en Packages | Etiqueta o registro mal escritos | Revisa `registry` y `tags` |
| `docker pull` dice *denied* | El paquete es privado | Autentícate con un token de solo lectura o hazlo público |
| Error por nombre de imagen no válido | `github.repository` contiene mayúsculas y GHCR exige minúsculas | Pasa el nombre a minúsculas antes de usarlo en `tags` |

## Autoevaluación

??? question "1. ¿Por qué el trabajo `test` se ejecuta antes que la publicación?"
    Porque `build-and-push` declara `needs: test`. Si las pruebas fallan, el segundo trabajo no se ejecuta, así que nunca se publica una imagen con código roto.

??? question "2. ¿Por qué un pull request no publica la imagen?"
    Porque la condición `if` exige que el evento sea un `push` a `main`. Un PR solo ejecuta las pruebas, que es lo que se quiere para validar sin publicar.

??? question "3. ¿De dónde sale el token con el que se publica la imagen?"
    De `secrets.GITHUB_TOKEN`, un token que GitHub genera automáticamente en cada ejecución. No hay ninguna credencial escrita en el YAML.

??? question "4. ¿Qué identifica a la imagen publicada y por qué importa?"
    El hash del commit (`github.sha`) usado como etiqueta. Permite saber exactamente qué código generó cada artefacto y volver a una versión concreta.

??? question "5. ¿Por qué `permissions` se limita al mínimo?"
    Para que el trabajo solo tenga los permisos necesarios (leer el código y escribir paquetes). Si algo saliera mal o se manipulara el *workflow*, el daño posible sería menor.

??? question "6. ¿Para qué sirve `workflow_dispatch`?"
    Para poder lanzar el *workflow* manualmente desde la pestaña Actions, sin necesidad de hacer un `push`.

??? question "7. En el workflow de esta web, ¿por qué no se ejecuta con los pull requests?"
    Porque `on` solo incluye `push` a `main`. La web publicada debe reflejar lo que ya está aprobado y fusionado, no lo que se propone en un PR.

## Recursos

- [Práctica original del curso](https://dpl-f1facc.gitlab.io/ut01/practicas/guiada-25-github-actions-cicd/)
- [Documentación de GitHub Actions](https://docs.github.com/es/actions)
- [Teoría de CI/CD](teoria.md)
