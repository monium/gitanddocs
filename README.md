# Uv y Zensical

Guía para instalar `uv`, configurar **Zensical**, levantar el servidor de desarrollo y aprovechar la recarga en tiempo real.

---

## Paso 1: Instalar `uv`

`uv` es un gestor de paquetes y entornos de Python, abre tu terminal y ejecuta:

```
curl -sSf https://astral.sh/uv/install.sh | sh
```

Tras la instalación, normalmente se recomienda cerrar y volver a abrir la terminal para que el comando `uv` esté disponible. Sin embargo, no es necesario reiniciar la terminal.

Puedes recargar la configuración de Bash en la terminal actual ejecutando:

```bash
source ~/.bashrc
```

Después, comprueba que `uv` está disponible:

`uv --version`

También puedes comprobar dónde está instalado el ejecutable con:

`which uv`

La ruta debería apuntar habitualmente a una ubicación como:

`~/.local/bin/uv`

De esta forma, puedes empezar a utilizar `uv` inmediatamente sin cerrar ni volver a abrir la terminal.

---

## Paso 2: Crear la carpeta del proyecto e inicializarlo

Crea un directorio para tu proyecto y entra en él:

```
mkdir mi-proyecto-docs
cd mi-proyecto-docs
```

A continuación, inicializa el proyecto con `uv`:

```
uv init
```

---

## Paso 3: Añadir la dependencia de Zensical

Instala **Zensical** dentro de tu proyecto mediante `uv`:

```
uv add zensical
```

`uv` gestionará automáticamente la descarga e instalación del paquete en el entorno virtual del proyecto.

> **Nota:** Para asegurarte de que Zensical solo se instale como herramienta de desarrollo (y no en el entorno de producción), utiliza la bandera `--dev`:

```
uv add --dev zensical
```

*Esto registrará `zensical` bajo el bloque `[tool.uv.dev-dependencies]` o `[dependency-groups]` en tu archivo `pyproject.toml`.*

---

## Paso 4: Levantar el servidor local

Para compilar la documentación y poner en marcha el servidor web de desarrollo, ejecuta:

```
uv run zensical serve
```

Verás una salida en la terminal indicando la dirección URL local, normalmente:

http://127.0.0.1:8000

Abre esa URL en tu navegador preferido para ver el sitio web generado.

---

## Paso 5: Edición y actualización en tiempo real (Live Reload)

El servidor de desarrollo de Zensical cuenta con un motor de recarga en tiempo real:

1. **Abre y edita:** Modifica cualquier archivo `.md` en tu editor de código.
2. **Guarda los cambios:** Al guardar el archivo, el servidor detectará la modificación de inmediato.
3. **Sincronización instantánea:** La página web abierta en el navegador reflejará los cambios automáticamente sin necesidad de recargar manualmente la página.
