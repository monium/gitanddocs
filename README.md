# Uv y Zensical

![Python](https://img.shields.io/badge/Python-3.14+-blue)
![Zensical](https://img.shields.io/badge/Zensical-0.0.65+-green)

Guía para instalar `uv`, configurar **Zensical**, levantar el servidor de desarrollo y aprovechar la recarga en tiempo real.

## ¿Qué son estas herramientas?

- **[uv](https://github.com/astral-sh/uv)**: Gestor de paquetes y entornos de Python ultrarrápido, escrito en Rust. Reemplaza a `pip`, `pip-tools` y `virtualenv` con una sola herramienta.
- **[Zensical](https://zensical.org/)**: Generador de sitios estáticos moderno, creado por el equipo de Material for MkDocs. Optimizado para documentación técnica con Markdown, soporta admonitions, tabs, diagramas Mermaid, matemáticas y más.

Juntos permiten crear documentación profesional con recarga en tiempo real y gestión moderna de dependencias.

---

## Índice

1. [Prerrequisitos](#prerrequisitos)
2. [Instalar uv](#paso-1-instalar-uv)
3. [Crear proyecto](#paso-2-crear-la-carpeta-del-proyecto-e-inicializarlo)
4. [Instalar Zensical](#paso-3-añadir-la-dependencia-de-zensical)
5. [Servidor de desarrollo](#paso-4-levantar-el-servidor-local)
6. [Live Reload](#paso-5-edición-y-actualización-en-tiempo-real-live-reload)
7. [Comandos útiles](#comandos-útiles)
8. [Estructura del proyecto](#estructura-del-proyecto)
9. [Solución de problemas](#solución-de-problemas)
10. [Recursos adicionales](#recursos-adicionales)

---

## Prerrequisitos

- Terminal con acceso a internet
- Git instalado (opcional, para control de versiones)
- Editor de código (VS Code, Sublime, Intellij IDEA etc.)

---

## Paso 1: Instalar `uv`

`uv` es un gestor de paquetes y entornos de Python. Abre tu terminal y ejecuta:

```bash
curl -sSf https://astral.sh/uv/install.sh | sh
```

Tras la instalación, normalmente se recomienda cerrar y volver a abrir la terminal para que el comando `uv` esté disponible. Sin embargo, no es necesario reiniciar la terminal.

**Comprobar la instalación:**

```bash
which uv
uv --version
```

*(Debería apuntar habitualmente a `~/.cargo/bin/uv` o `~/.local/bin/uv`)*

---

## Paso 2: Crear la carpeta del proyecto e inicializarlo

Crea un directorio para tu proyecto y entra en él:

```bash
mkdir mi-proyecto-docs
cd mi-proyecto-docs
```

A continuación, inicializa el proyecto con `uv`:

```bash
uv init
```

Esto crea la estructura básica del proyecto, incluyendo `pyproject.toml` y el archivo `.python-version`.

---

## Paso 3: Añadir la dependencia de Zensical

Instala **Zensical** como dependencia de desarrollo (recomendado):

```bash
uv add --dev zensical
```

Esto registra `zensical` bajo `[dependency-groups.dev]` en `pyproject.toml`, indicando que solo se necesita para desarrollo, no para producción.

`uv` gestionará automáticamente la descarga e instalación del paquete en el entorno virtual del proyecto.

---

## Paso 4: Levantar el servidor local

Para compilar la documentación y poner en marcha el servidor web de desarrollo, ejecuta:

```bash
uv run zensical serve
```

Verás una salida en la terminal indicando la dirección URL local, normalmente:

```
http://127.0.0.1:8000
```

Abre esa URL en tu navegador preferido para ver el sitio web generado.

---

## Paso 5: Edición y actualización en tiempo real (Live Reload)

El servidor de desarrollo de Zensical cuenta con un motor de recarga en tiempo real:

1. **Abre y edita:** Modifica cualquier archivo `.md` en tu editor de código.
2. **Guarda los cambios:** Al guardar el archivo, el servidor detectará la modificación de inmediato.
3. **Sincronización instantánea:** La página web abierta en el navegador reflejará los cambios automáticamente sin necesidad de recargar manualmente la página.

---

## Comandos útiles

| Comando | Descripción |
|---------|-------------|
| `uv run zensical serve` | Inicia servidor de desarrollo con live reload |
| `uv run zensical build` | Compila el sitio para producción |
| `uv run zensical build --clean` | Compila limpiando la carpeta `site` primero |
| `uv add --dev zensical` | Añade Zensical como dependencia de desarrollo |
| `uv remove zensical` | Elimina Zensical del proyecto |
| `uv run zensical --help` | Muestra todos los comandos disponibles |

---

## Estructura del proyecto

Tras seguir esta guía, tu proyecto tendrá esta estructura:

```
mi-proyecto-docs/
├── docs/
│   ├── index.md      # Página principal
│   └── markdown.md   # Página de ejemplo
├── src/
│   └── tu_paquete/
├── .python-version
├── pyproject.toml
├── README.md
└── zensical.toml     # Configuración de Zensical
```

---

## Solución de problemas

### El comando `uv` no se encuentra

Asegúrate de haber cerrado y reabierto la terminal tras instalar uv. Si persiste:

```bash
# Verifica la instalación
which uv

# Si usas bash/zsh, añade al ~/.bashrc o ~/.zshrc:
export PATH="$HOME/.cargo/bin:$PATH"
```

### Error de puerto ocupado

Si el puerto 8000 está en uso, Zensical usará otro automáticamente. También puedes especificarlo:

```bash
uv run zensical serve --dev-addr 0.0.0.0:8080
```

### Zensical no detecta cambios

Asegúrate de:
- Estar editando archivos dentro de la carpeta `docs/`
- Guardar los cambios (Ctrl+S / Cmd+S)
- Que el servidor se ejecutó con `uv run zensical serve` (no `build`)

---

## Recursos adicionales

- [Documentación oficial de Zensical](https://zensical.org/docs/)
- [Repositorio de Zensical en GitHub](https://github.com/zensical/zensical)
- [Documentación de uv](https://docs.astral.sh/uv/)
- [Guía de migración desde MkDocs](https://zensical.org/docs/compatibility/mkdocs/migration/)
- [Características de Zensical](https://zensical.org/studio/features/)