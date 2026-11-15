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

