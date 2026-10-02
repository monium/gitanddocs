---
icon: lucide/target
---

# Cómo obtener un site con Zensical
Seguramente te estarás preguntando cómo convertir tu markdown a un site HTML/CSS/JS completo. Pues en tan solo **2** sencillos pasos te explicaré cómo.

!!! warning

    Dando por hecho que ya has creado el proyecto con uv y zensical y que te encuentras en el directorio /docs

- Primero editamos / creamos los archivos markdown que encontraremos en nuestro directorio. Estos `.md` serán los que **zensical** use para crear el site.

<br>

- Seguidamente, para obtener nuestro site en formato "árbol de directorios lleno de HTML/CSS/JS que podamos subir y usar" solo escribiremos `uv run zensical build`, que generará el site en la carpeta `site/`, dentro de la raíz del proyecto.

<br>

**¡Y listo!** ya lo tendrías, pero quédate si quieres saber cómo ver un preview del site en tu localhost.

<br>

- Abrimos nuestra terminal para ejecutar el siguiente comando:
```bash
uv run zensical serve
```
Esto abrirá un servidor en nuestro **localhost**, por defecto en el puerto **8000**.

!!! note

    Para cambiar el puerto en caso de que esté ocupado usamos -a e indicamos el puerto. 

    Ejemplo: `-a localhost:8001`

<br>

-  Por último esto debería dejar en terminal el mensaje **"No issues found"** y el servidor corriendo en local. Solo entonces abrimos nuestro navegador a `http://localhost:8000` y entramos en la vista previa de nuestro site.
