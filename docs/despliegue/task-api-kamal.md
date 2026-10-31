---
icon: lucide/rocket
---

# Desplegar una API con Docker y Kamal (task-api)

Caso práctico visto en clase: una API de tareas hecha con **FastAPI** y **SQLite**, empaquetada en **Docker** y desplegada por SSH en servidores propios con **Kamal**.

Repositorio de ejemplo: [monium/task-api](https://github.com/monium/task-api)

## ¿Qué vamos a conseguir?

Pasar de "funciona en mi ordenador" a "funciona en un servidor, con HTTPS, y puedo publicar una versión nueva con un solo comando".

```text
Tu ordenador ──(1) docker build──► Imagen
     │                               │
     │                        (2) push al registro (ghcr.io)
     │                               ▼
     └──(3) kamal deploy por SSH──► Servidor ──► kamal-proxy ──► contenedor de la API
```

## Herramientas

| Herramienta | Para qué sirve |
|---|---|
| **FastAPI** | Framework web de Python con el que está hecha la API. |
| **SQLAlchemy + SQLite** | Acceso a datos y base de datos en un único archivo. |
| **uv** | Gestor de paquetes y entorno de Python (ya lo usamos con Zensical). |
| **Docker** | Empaqueta la aplicación y sus dependencias en una imagen. |
| **Kamal** | Despliega esa imagen en uno o varios servidores por SSH, sin tiempo de caída. |
| **just** | Lanzador de comandos: guarda recetas cortas en un `justfile`. |
| **ghcr.io** | Registro de imágenes de GitHub donde se sube la imagen. |

## Estructura del proyecto

```text
task-api/
├── app/
│   ├── main.py        # rutas de la API
│   ├── models.py      # tabla Task (SQLAlchemy)
│   ├── schemas.py     # validación de entrada y salida (Pydantic)
│   └── database.py    # conexión a la base de datos
├── config/
│   ├── deploy.yml             # configuración común de Kamal
│   ├── deploy.staging.yml     # solo para staging
│   └── deploy.production.yml  # solo para producción
├── scripts/
│   ├── deploy.sh
│   └── export-requirements.sh
├── Dockerfile
├── justfile
├── pyproject.toml
└── uv.lock
```

## La API

Es un CRUD de tareas. Cada tarea tiene `id`, `title`, `description`, `completed` y `created_at`.

| Método | Ruta | Qué hace |
|---|---|---|
| `POST` | `/tasks` | Crea una tarea (devuelve `201`). |
| `GET` | `/tasks` | Lista tareas con `skip` y `limit` (máximo 100). |
| `GET` | `/tasks/{id}` | Devuelve una tarea o `404`. |
| `PATCH` | `/tasks/{id}` | Modifica solo los campos enviados. |
| `DELETE` | `/tasks/{id}` | Borra la tarea (devuelve `204`). |
| `GET` | `/health` | Comprobación sencilla de que la API responde. |
| `GET` | `/up` | Comprobación que usa Kamal antes de dar por buena una versión. |

!!! note "¿Por qué existe `/up`?"
    Kamal considera que un contenedor está sano cuando `/up` responde `200`. Solo entonces le pasa el tráfico. Si una versión nueva falla, el tráfico sigue en la anterior.

### Probarla en local

```bash
uv sync
uv run uvicorn app.main:app --reload
```

FastAPI genera documentación interactiva en `http://localhost:8000/docs`.

La base de datos se elige con la variable `DATABASE_URL`. Si no existe, usa `sqlite:///./tasks.db`.

