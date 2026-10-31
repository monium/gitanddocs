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

## Paso 1: el Dockerfile

```dockerfile
FROM python:3.14-slim

WORKDIR /app

COPY requirements.txt .
RUN python -m pip install --no-cache-dir -r requirements.txt

COPY app/ ./app/

RUN useradd --create-home --uid 10001 appuser \
    && mkdir -p /data \
    && chown -R appuser:appuser /app /data
USER 10001

EXPOSE 8000
CMD ["python", "-m", "uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

Decisiones importantes:

- **Primero `requirements.txt`, después el código.** Docker guarda cada instrucción como una capa en caché. Si solo cambias código, no se reinstalan las dependencias.
- **Usuario sin privilegios.** El proceso no corre como `root`, así que un fallo de seguridad en la app hace menos daño.
- **`--host 0.0.0.0`.** Dentro de un contenedor hay que escuchar en todas las interfaces; si no, nadie podría conectarse desde fuera.
- **`/data` para SQLite.** La base de datos vive en un volumen, no dentro de la imagen.

### `.dockerignore`

Evita que entren en la imagen archivos que no deben: `.git`, `.venv`, `.env*`, `*.db`, cachés. Así la imagen es más pequeña y no se cuelan secretos.

### Exportar dependencias

El `Dockerfile` instala desde `requirements.txt`, que se genera a partir del `uv.lock`:

```bash
uv export --locked --no-dev --no-emit-project --format requirements.txt --output-file requirements.txt
```

- `--locked`: falla si el `uv.lock` no está al día.
- `--no-dev`: no incluye `pytest` ni herramientas de desarrollo.

## Paso 2: probar la imagen en local

```bash
docker build -t tasks-api:local .
docker volume create tasks-api-data
docker run --rm --name tasks-api -p 8000:8000 \
  -v tasks-api-data:/data \
  -e DATABASE_URL=sqlite:////data/tasks.db \
  tasks-api:local
```

Se usa un **volumen** para que las tareas no se pierdan al parar el contenedor.

!!! tip "Cuatro barras"
    `sqlite:////data/tasks.db` lleva cuatro `/`: tres del formato de la URL y una de la ruta absoluta `/data/tasks.db`.

## Paso 3: el `justfile`

Para no recordar comandos largos, se guardan como recetas:

| Receta | Qué hace |
|---|---|
| `just setup-local` | Instala dependencias (`uv sync`). |
| `just run` | Arranca la API con recarga automática. |
| `just test` | Ejecuta `pytest`. |
| `just docker-build` | Exporta dependencias y construye la imagen. |
| `just docker-run` | Arranca la imagen en local con volumen. |
| `just config staging` | Valida la configuración de Kamal. |
| `just setup staging` | Primer despliegue (prepara el servidor). |
| `just deploy staging` | Despliega una versión nueva. |
| `just status staging` | Muestra el estado de los servicios. |
| `just logs staging` | Muestra los logs de la aplicación. |

El destino por defecto es `staging`. Las recetas rechazan cualquier destino que no sea `staging` o `production`, así un error de escritura no despliega donde no toca.

## Paso 4: configurar Kamal

### Qué es Kamal

Kamal entra por SSH al servidor, instala Docker si hace falta, descarga tu imagen y arranca el contenedor detrás de un proxy (**kamal-proxy**). Al publicar una versión nueva arranca primero el contenedor nuevo, espera a que `/up` responda bien y solo entonces retira el viejo. Por eso no hay corte.

### Configuración común: `config/deploy.yml`

```yaml
service: tasks-api

image: usuario/tasks-api

registry:
  server: ghcr.io
  username: usuario
  password:
    - KAMAL_REGISTRY_PASSWORD

ssh:
  user: ubuntu

builder:
  arch: amd64

proxy:
  app_port: 8000

env:
  clear:
    PYTHONUNBUFFERED: "1"
```

| Clave | Significado |
|---|---|
| `service` | Nombre de la aplicación en el servidor. |
| `image` | Nombre de la imagen en el registro. |
| `registry` | Dónde se sube y se descarga la imagen. |
| `password` | **No se escribe aquí.** Es el nombre de una variable secreta. |
| `ssh.user` | Usuario con el que Kamal entra al servidor. |
| `builder.arch` | Arquitectura del servidor. Si construyes en un Mac con chip ARM y el servidor es `amd64`, hay que indicarlo. |
| `proxy.app_port` | Puerto en el que escucha la app dentro del contenedor. |
| `env.clear` | Variables de entorno no secretas. |

!!! danger "Secretos"
    El token del registro (`KAMAL_REGISTRY_PASSWORD`) va en un archivo de secretos que **no se sube a Git**. Si lo subes por error, hay que revocarlo.

### Un archivo por entorno

Kamal llama **destino** a cada entorno. Con `-d staging`, mezcla `deploy.yml` con `deploy.staging.yml`.

`config/deploy.staging.yml`:

```yaml
servers:
  web:
    - 203.0.113.10        # IP del servidor de staging

proxy:
  host: stg.example.com
  app_port: 8000
  ssl: true

volumes:
  - /var/lib/tasks-api:/data

env:
  clear:
    APP_ENV: staging
    DATABASE_URL: sqlite:////data/tasks.db
```

- `servers.web`: máquinas donde se despliega.
- `proxy.host` + `ssl: true`: el proxy pide un certificado **Let's Encrypt** automáticamente para ese dominio.
- `volumes`: monta una carpeta del servidor en `/data` del contenedor. Ahí vive SQLite.

!!! warning "Requisitos para que funcione el HTTPS"
    El dominio debe apuntar por DNS a la IP del servidor, y los puertos 80 y 443 deben estar abiertos.

!!! warning "Permisos del volumen"
    El contenedor corre con el usuario `10001`. La carpeta `/var/lib/tasks-api` del servidor debe poder escribirla ese usuario; si no, SQLite fallará al crear la base de datos.

## Paso 5: desplegar

Orden habitual:

```bash
just config staging     # comprueba que la configuración es válida
just setup staging      # SOLO la primera vez
just deploy staging     # las siguientes veces
just status staging     # ¿está todo en marcha?
just logs staging       # ¿qué dice la aplicación?
```

Qué hace `setup` frente a `deploy`:

| Comando | Cuándo | Qué hace |
|---|---|---|
| `kamal setup` | Primera vez | Prepara el servidor (Docker, proxy), construye, sube y arranca. |
| `kamal deploy` | Cada versión nueva | Construye, sube, arranca la nueva y retira la vieja. |

Una vez desplegado:

```bash
curl https://stg.example.com/health
curl -X POST https://stg.example.com/tasks \
  -H "Content-Type: application/json" \
  -d '{"title": "Primera tarea"}'
```

### Volver a una versión anterior

```bash
kamal rollback --help
```

Kamal conserva las imágenes anteriores, así que puedes volver a una versión que funcionaba indicando su identificador.

### El script `scripts/deploy.sh`

Es lo mismo que la receta `deploy`, pero como script de Bash:

```bash
./scripts/deploy.sh staging           # despliega
./scripts/deploy.sh production setup  # primer despliegue en producción
```

Usa `set -Eeuo pipefail`: se detiene ante cualquier error, variable sin definir o fallo dentro de una tubería. Valida el destino y exporta las dependencias **antes** de llamar a Kamal.

## Buenas prácticas y fallos típicos

- **No versiones la base de datos.** Un `tasks.db` dentro del repositorio indica que falta una línea en `.gitignore` (por ejemplo `*.db`).
- **Revisa cada entorno.** Si copias `deploy.staging.yml` a `deploy.production.yml`, cambia `APP_ENV`, el servidor y el dominio. Es fácil dejar un valor de staging en producción.
- **Prueba siempre en staging antes de producción.**
- **No pongas IPs ni dominios reales en documentación pública** si no hace falta.
- **SQLite es válido para aprender o para poca carga.** Si hay varios servidores o mucho tráfico, conviene PostgreSQL.

| Síntoma | Causa probable |
|---|---|
| Kamal no puede conectar | Usuario SSH incorrecto o clave no autorizada en el servidor. |
| `denied` al subir la imagen | Token del registro ausente o sin permiso de escritura. |
| La versión nueva no pasa a producción | `/up` no responde `200`; mira `just logs`. |
| No se emite el certificado | El DNS no apunta al servidor o los puertos 80/443 están cerrados. |
| Error de arquitectura al arrancar | Falta `builder.arch` o no coincide con el servidor. |
| SQLite no puede escribir | Permisos de la carpeta montada. |

## Autoevaluación

??? question "1. ¿Por qué se copia `requirements.txt` antes que el código en el Dockerfile?"
    Para aprovechar la caché de capas de Docker. Si solo cambia el código, la capa de dependencias no se reconstruye y el build es mucho más rápido.

??? question "2. ¿Por qué la aplicación escucha en `0.0.0.0` dentro del contenedor?"
    Porque con `127.0.0.1` solo aceptaría conexiones desde dentro del propio contenedor. Con `0.0.0.0` acepta las que llegan a través de la red de Docker, que es como llega el proxy.

??? question "3. ¿Para qué sirve el volumen montado en `/data`?"
    Para que la base de datos SQLite persista. Los datos del sistema de archivos de un contenedor se pierden al reemplazarlo; el volumen vive en el servidor y sobrevive a los despliegues.

??? question "4. ¿Qué diferencia hay entre `kamal setup` y `kamal deploy`?"
    `setup` se usa la primera vez: además de desplegar, prepara el servidor (instala Docker y arranca el proxy). `deploy` solo publica una versión nueva en un servidor ya preparado.

??? question "5. ¿Cómo evita Kamal cortes al publicar una versión nueva?"
    Arranca el contenedor nuevo junto al antiguo, espera a que su comprobación de salud (`/up`) responda correctamente, mueve el tráfico y entonces retira el antiguo. Si la comprobación falla, el antiguo sigue sirviendo.

??? question "6. ¿Por qué `KAMAL_REGISTRY_PASSWORD` no aparece con su valor en `deploy.yml`?"
    Porque es un secreto. En el YAML solo está el nombre de la variable; el valor se guarda en un archivo de secretos que no se sube a Git.

??? question "7. ¿Qué hace `ssl: true` junto con `proxy.host`?"
    Hace que kamal-proxy obtenga y renueve un certificado Let's Encrypt para ese dominio. Para ello el DNS debe apuntar al servidor y los puertos 80 y 443 deben estar abiertos.

??? question "8. ¿Por qué el contenedor usa un usuario sin privilegios?"
    Para limitar el daño si alguien consigue ejecutar código en la aplicación: no tendría permisos de `root` dentro del contenedor.

??? question "9. ¿Para qué sirven staging y production por separado?"
    Staging permite probar una versión en un entorno parecido al real sin afectar a los usuarios. Cuando funciona, se despliega a producción con los mismos pasos.

??? question "10. ¿Qué significa `builder.arch: amd64`?"
    Que la imagen se construye para procesadores x86-64. Es necesario si se construye desde una máquina con arquitectura distinta (por ejemplo, un Mac con chip ARM) y el servidor es `amd64`.

## Recursos

- Repositorio: [monium/task-api](https://github.com/monium/task-api)
- [Documentación de Kamal](https://kamal-deploy.org/)
- [Documentación de FastAPI](https://fastapi.tiangolo.com/)
- [Documentación de uv](https://docs.astral.sh/uv/)
- [just](https://github.com/casey/just)
