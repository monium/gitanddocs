---
icon: simple/gitea
---


# Instalar Gitea con Docker

Guía paso a paso para montar tu propio servidor de Git (**Gitea**) usando **Docker**, con dos opciones de base de datos: **SQLite3** (la que ya viene incluida) o **PostgreSQL**. No hace falta saber de informática: cada paso explica **qué haces, por qué y qué está pasando por dentro**.

---

## 1. ¿Qué es Gitea?

Si conoces GitHub, Gitea es lo mismo pero **instalado en tu propia máquina**. Sirve para guardar tus proyectos con Git, ver su historial y trabajar con otras personas, y todo queda en tu servidor, no en uno de fuera.

Tiene una web desde la que se crean usuarios y repositorios, se ven los cambios y se gestionan incidencias.

---

## 2. ¿Qué es Docker?

Instalar un programa normal implica crear usuarios, carpetas, permisos y servicios a mano. Docker lo simplifica: el programa viaja dentro de una **caja cerrada** que ya trae todo lo que necesita, y tú solo le dices a Docker "ejecuta esta caja".

### La analogía del táper

| Instalación normal | Con Docker |
|---|---|
| Cocinar en tu cocina: compras ingredientes, preparas la mesa y lo cocinas | Un táper ya preparado: lo abres y lo calientas |
| Todo queda mezclado con tu sistema | Todo queda aislado en la caja |
| Para quitarlo hay que deshacer cada paso | Se borra la caja y listo |

### Palabras que vas a ver

| Palabra | En sencillo |
|---|---|
| **Docker Engine** | El programa que ejecuta las cajas |
| **Imagen** | La caja cerrada con el programa dentro (por ejemplo, Gitea) |
| **Contenedor** | Una imagen en marcha |
| **Docker Compose** | Una herramienta que arranca contenedores leyendo un archivo de texto, el `docker-compose.yml` |
| **Volumen** | Una carpeta de tu máquina conectada a la caja, para que los datos **no se pierdan** al borrarla |
| **Puerto** | El número de una "ventanilla" por la que se accede (Gitea usa la 3000) |
| **Rootless** | Versión de la imagen que **no necesita permisos de administrador** dentro de la caja. Es más segura |

---

## 3. Qué necesitas antes de empezar

- Una máquina con **Ubuntu** (22.04, 24.04 o 26.04, en 64 bits).
- Conexión a internet.
- Un usuario con permisos de `sudo`.

Comprueba tu sistema:

```bash
lsb_release -a
uname -m
```

El segundo comando debe responder `x86_64`.

---

## 4. Instalar Docker y Docker Compose

Primero comprueba si ya los tienes:

```bash
docker --version
docker compose version
```

Si responden con un número de versión, **salta al apartado 5**. Si dicen que no encuentran la orden, instálalos.

### Versión corta (suficiente para practicar)

```bash
sudo apt update
sudo apt install docker.io docker-compose-v2
sudo systemctl status docker
sudo docker run hello-world
```

**Qué hace cada comando:**

| Comando | Qué hace |
|---|---|
| `apt update` | Actualiza la lista de programas disponibles |
| `apt install docker.io docker-compose-v2` | Instala Docker y el comando `docker compose` |
| `systemctl status docker` | Pregunta si Docker está encendido. Debe poner `active (running)` (sal con `q`) |
| `docker run hello-world` | Descarga una caja de prueba y la ejecuta |

!!! warning "Deja que `hello-world` termine"
    La primera vez descarga la imagen y puede tardar unos segundos. Si pulsas `Ctrl + C` se cancela. Debe acabar mostrando un mensaje que empieza por `Hello from Docker!`.

!!! note "No instales `docker`"
    En Ubuntu, `sudo apt install docker` instala **otro programa distinto** que no tiene nada que ver con los contenedores. El paquete correcto se llama `docker.io`.

### Versión oficial (la que recomienda Docker)

El paquete `docker.io` es el que trae Ubuntu en su catálogo, con una versión algo más antigua. Docker también ofrece su repositorio oficial, con la última versión. Los pasos están en <https://docs.docker.com/engine/install/ubuntu/>.

Las dos versiones **chocan entre sí**: elige una y no las mezcles.

### Comprobar que todo quedó bien

```bash
docker --version
docker compose version
```

Ejemplo de respuesta:

```
Docker version 29.1.3, build 29.1.3-0ubuntu3~24.04.2
Docker Compose version 2.40.3+ds1-0ubuntu1~24.04.1
```

### ¿Por qué escribo `sudo` delante de Docker?

Docker se controla por un archivo especial al que solo pueden acceder el administrador y el grupo `docker`. Si escribes `docker ps` sin `sudo`, verás:

```
permission denied while trying to connect to the docker API at unix:///var/run/docker.sock
```

Es normal y tiene dos soluciones:

- **Seguir usando `sudo`** delante de cada comando de Docker. Es lo que hace esta guía.
- **Añadir tu usuario al grupo `docker`** y volver a iniciar sesión:

```bash
sudo usermod -aG docker $USER
```

!!! warning "Aviso de seguridad"
    Pertenecer al grupo `docker` equivale, en la práctica, a tener permisos de administrador sobre la máquina. En una VM de prácticas no importa, pero conviene saberlo.

---

## 5. Elegir la base de datos

Gitea necesita una base de datos para guardar usuarios, repositorios e incidencias. Hay dos opciones:

| | SQLite3 | PostgreSQL |
|---|---|---|
| **Dónde vive** | Un archivo dentro de Gitea | Un servidor de base de datos independiente |
| **Qué hay que instalar** | Nada, ya viene incluida | Un contenedor más |
| **Complejidad** | Mínima | Algo mayor |
| **Cuándo usarla** | Pruebas, uso personal, pocos usuarios | Muchos usuarios, uso profesional |
| **Contenedores** | 1 (Gitea) | 2 (Gitea y PostgreSQL) |

!!! tip "Elígela antes de terminar el asistente"
    Cambiar de base de datos **después** de crear usuarios implica migrar los datos. Decide ahora.

Los pasos 6, 7 y 8 son iguales en los dos casos, salvo el archivo del paso 7. El paso 9 (el asistente) tiene una tabla para cada base de datos.

---

## 6. Crear la carpeta del proyecto

```bash
mkdir -p ~/gitea/{data,config}
cd ~/gitea
```

Dentro de `gitea` se crean dos carpetas:

| Carpeta | Para qué sirve |
|---|---|
| `data` | Repositorios y datos de Gitea. **Lo que no debe perderse** |
| `config` | El archivo de configuración (`app.ini`) |

Son carpetas de **tu máquina** que se conectan a la caja. Aunque borres el contenedor, tus datos siguen aquí.

### Permisos

La imagen *rootless* usa por defecto el usuario número **1000**, y las dos carpetas tienen que pertenecerle. Comprueba tu número:

```bash
id -u
```

- Si responde **1000** (lo normal en el primer usuario de Ubuntu), las carpetas que has creado ya son tuyas: no hay que hacer nada.
- Si responde **otro número**, ejecuta:

```bash
sudo chown 1000:1000 config data
```

!!! warning "Si te saltas esto cuando hace falta"
    El contenedor falla con un error `Permission denied` en sus registros, por ejemplo `/var/lib/gitea/git is not writable`.

---

## 7. Crear el archivo `docker-compose.yml`

Este archivo **describe** las cajas que quieres: qué imagen usar, qué carpetas conectar y qué puertos abrir. Elige **solo uno** de los dos.

### Opción A: con SQLite3

```bash
cat > docker-compose.yml <<'EOF'
services:
  server:
    image: docker.gitea.com/gitea:28.1.0-rootless
    restart: always
    volumes:
      - ./data:/var/lib/gitea
      - ./config:/etc/gitea
      - /etc/timezone:/etc/timezone:ro
      - /etc/localtime:/etc/localtime:ro
    ports:
      - "3000:3000"
      - "2222:2222"
EOF
```

#### Qué significa cada línea

| Línea | En sencillo |
|---|---|
| `image` | Qué caja usar: Gitea, en la versión `28.1.0`, edición *rootless* |
| `restart: always` | Si se apaga o reinicias la máquina, vuelve a encenderse sola |
| `volumes` | Carpetas de tu máquina conectadas a la caja. El `:ro` significa "solo lectura" |
| `./data:/var/lib/gitea` | Tu carpeta `data` es la carpeta de datos dentro de la caja |
| `./config:/etc/gitea` | Tu carpeta `config` es la de configuración dentro de la caja |
| `/etc/timezone` y `/etc/localtime` | La caja usa la misma hora que tu máquina |
| `"3000:3000"` | La puerta 3000 de tu máquina lleva a la web de Gitea |
| `"2222:2222"` | La puerta 2222 lleva al SSH de Gitea, para `git push` por SSH |

### Opción B: con PostgreSQL

```bash
cat > docker-compose.yml <<'EOF'
networks:
  gitea:
    external: false

services:
  server:
    image: docker.gitea.com/gitea:28.1.0-rootless
    restart: always
    environment:
      - GITEA__database__DB_TYPE=postgres
      - GITEA__database__HOST=db:5432
      - GITEA__database__NAME=tu-nombre
      - GITEA__database__USER=tu-usuario
      - GITEA__database__PASSWD=tu-password
    networks:
      - gitea
    volumes:
      - ./data:/var/lib/gitea
      - ./config:/etc/gitea
      - /etc/timezone:/etc/timezone:ro
      - /etc/localtime:/etc/localtime:ro
    ports:
      - "3000:3000"
      - "2222:2222"
    depends_on:
      - db

  db:
    image: docker.io/library/postgres:18
    restart: always
    environment:
      - POSTGRES_USER=tu-usuario
      - POSTGRES_PASSWORD=tu-contraseña
      - POSTGRES_DB=gitea
    networks:
      - gitea
    volumes:
      - ./postgres:/var/lib/postgresql
EOF
```

#### Qué cambia respecto a la opción A

Aparece un **segundo contenedor** (`db`) con PostgreSQL, y tres piezas nuevas:

| Parte | En sencillo |
|---|---|
| `networks: gitea` | Una red privada para que los dos contenedores se vean entre sí |
| `GITEA__database__...` | Le dicen a Gitea de antemano qué base de datos usar: tipo, dónde está, nombre, usuario y contraseña |
| `HOST=db:5432` | `db` es el nombre del otro contenedor y `5432` el puerto estándar de PostgreSQL |
| `depends_on: db` | Arranca primero la base de datos y después Gitea |
| `POSTGRES_USER/PASSWORD/DB` | Al arrancar, crea el usuario, la contraseña y la base de datos `gitea` |
| `./postgres:/var/lib/postgresql` | Carpeta de tu máquina donde PostgreSQL guarda sus datos |

!!! note "¿Por qué `db` y no `localhost`?"
    Dentro de una caja, `localhost` significa "esta misma caja". Los contenedores de una misma red se encuentran por el **nombre del servicio**, y por eso Gitea llama a la base de datos `db`.

!!! warning "Las contraseñas deben coincidir"
    Aquí todo es `gitea`, como en el ejemplo oficial, y sirve para practicar. Si cambias la contraseña, tiene que ser **la misma** en `POSTGRES_PASSWORD` y en `GITEA__database__PASSWD`.

!!! note "El puerto de la base de datos no se abre"
    El contenedor `db` no publica ningún puerto hacia fuera: solo Gitea puede hablar con PostgreSQL.

!!! note "Versión de PostgreSQL"
    La documentación de Gitea fija PostgreSQL en una versión mayor a propósito (aquí la 18), porque PostgreSQL no actualiza solo su carpeta de datos. No cambies de versión mayor más adelante sin migrar.

### Comprobar que el archivo quedó bien

```bash
cat docker-compose.yml
```

La indentación importa: en un `.yml` los espacios son parte del contenido. Si copiaste el bloque entero, estará bien.

---

## 8. Arrancar Gitea

```bash
sudo docker compose up -d
sudo docker compose ps
```

| Parte | Significado |
|---|---|
| `docker compose` | Usa el `docker-compose.yml` de esta carpeta |
| `up` | Crea y arranca lo que describe |
| `-d` | En segundo plano, para no ocupar la terminal |
| `ps` | Muestra los contenedores y su estado |

La primera vez descarga las imágenes, y puede tardar. No pulses `Ctrl + C`.

**Resultado correcto:** en `ps` aparece el contenedor (o los **dos**, con PostgreSQL) en estado `Up`. En la columna de puertos verás algo como `0.0.0.0:3000->3000/tcp`.

---

## 9. El asistente de instalación

Abre en el navegador:

```
http://127.0.0.1:3000
```

Verás la página **Configuración inicial**.

!!! warning "Si usas PostgreSQL, espera medio minuto"
    La base de datos necesita unos segundos para arrancar antes de que el asistente pueda conectarse.

### 9.1 Base de datos

**Con SQLite3:** en *Tipo de base de datos* elige **SQLite3** y deja el resto como viene. No hay nada más que configurar.

**Con PostgreSQL:** el asistente debería aparecer ya con los campos rellenos. Compruébalos:

| Campo | Valor |
|---|---|
| Tipo de base de datos | **PostgreSQL** |
| Servidor | **`db:5432`** |
| Nombre de usuario | **`tu-usuario`** |
| Contraseña | **`tu-contraseña`** (o la que hayas puesto) |
| Nombre de la base de datos | **`gitea`** |
| SSL | **Disable** |
| Esquema | Vacío |

!!! warning "Si ves `localhost:3306`"
    Eso es el puerto de MySQL y significa que **no** estás usando el archivo de PostgreSQL (opción B), o que has cambiado el tipo a mano. Con ese valor la instalación falla. Cierra el asistente, usa el archivo de la opción B y vuelve a empezar (apartado 13).

### 9.2 Configuración general

| Campo | Qué hacer |
|---|---|
| **Título del sitio** | El que quieras, por ejemplo `My Cup of Tea` |
| **Data Path** | Déjalo en `/var/lib/gitea` |
| **Ejecutar como usuario** | Déjalo en `git` (está bloqueado) |
| **Puerto de servidor SSH** | `2222` |
| **HTTP Server Port** | `3000` |
| **Gitea Website URL** | `http://localhost:3000/` por ahora |

### 9.3 Casillas

| Casilla | Recomendación |
|---|---|
| *Only administrators can create user accounts* | **Márcala**: nadie más podrá registrarse, los usuarios los creas tú |
| *Requerir CAPTCHA durante el registro* | Sin marcar |
| *Require sign-in to view pages* | Sin marcar, salvo que quieras que todo exija iniciar sesión |
| *Enable update checker* | Como esté |

### 9.4 Cuenta de administrador

Despliega **Configuración de la cuenta de administrador** y rellénala:

| Campo | Qué poner |
|---|---|
| Nombre de usuario | El tuyo. **No uses `admin`**: Gitea lo tiene reservado y lo rechaza |
| Correo electrónico | Uno con formato válido, por ejemplo `usuario@example.com` |
| Contraseña | Una que recuerdes |
| Confirmar contraseña | La misma |

Las secciones *Configuración de Correo* y *Configuración del servidor y de servicios de terceros* déjalas como están.

### 9.5 Instalar

Pulsa **Instalar Gitea**. Tarda unos segundos y la página puede parecer que se queda cargando: espera. Si no te lleva sola a la pantalla de inicio de sesión, entra en `http://127.0.0.1:3000` e inicia sesión con la cuenta que acabas de crear.

---

## 10. Comprobar que todo funciona

### Ver los contenedores

```bash
sudo docker compose ps
```

### Ver los registros

```bash
sudo docker compose logs --tail 20
```

Muestra las últimas 20 líneas de lo que ha escrito Gitea. Si algo falla, aquí aparece la causa.

### Comprobar la base de datos (solo con PostgreSQL)

```bash
sudo docker compose exec db psql -U gitea -d gitea -c '\dt'
```

| Parte | Significado |
|---|---|
| `exec db` | Ejecuta un comando dentro del contenedor `db` |
| `psql` | El cliente de línea de comandos de PostgreSQL |
| `-U gitea -d gitea` | Usuario `gitea`, base de datos `gitea` |
| `-c '\dt'` | Lista las tablas |

Debe aparecer una lista larga de tablas (`user`, `repository`, `issue`...). Es la prueba de que Gitea guarda sus datos en PostgreSQL.

### Ver dónde se guardan los datos

```bash
ls ~/gitea
ls ~/gitea/config
```

Verás las carpetas `data` y `config` (y `postgres`, si usas esa base de datos). En `config` aparecerá `app.ini`, el archivo de configuración que creó el asistente.

---

## 11. Comandos de Docker para el día a día

Todos se ejecutan **desde la carpeta `~/gitea`**.

| Comando | Qué hace |
|---|---|
| `sudo docker compose up -d` | Crea y arranca Gitea |
| `sudo docker compose stop` | Lo **para** sin borrar nada |
| `sudo docker compose start` | Lo vuelve a arrancar |
| `sudo docker compose restart` | Lo reinicia |
| `sudo docker compose down` | Para y **borra los contenedores** (los datos de `data`, `config` y `postgres` se quedan) |
| `sudo docker compose ps` | Muestra el estado |
| `sudo docker compose logs --tail 30` | Últimas 30 líneas de registro |
| `sudo docker compose logs -f` | Registros en directo (`Ctrl + C` para salir) |
| `sudo docker compose pull` | Descarga versiones nuevas de las imágenes |

### Comandos de Docker en general

| Comando | Qué hace |
|---|---|
| `sudo docker ps` | Lista los contenedores en marcha |
| `sudo docker ps -a` | Lista también los parados |
| `sudo docker images` | Lista las imágenes descargadas |
| `sudo systemctl status docker` | Estado del servicio Docker |
| `sudo systemctl stop docker` | Apaga Docker (y sus contenedores) |
| `sudo systemctl start docker` | Enciende Docker |

### Diferencia entre `stop` y `down`

| | `stop` | `down` |
|---|---|---|
| **Contenedor** | Se queda, pero parado | Se borra |
| **Tus datos** | Intactos | Intactos |
| **Para volver** | `start` | `up -d` |

Es como apagar la tele con el mando (`stop`) o desenchufarla y guardarla en el trastero (`down`). En los dos casos tus cosas siguen en las carpetas `data` y `config`.

---

## 12. Actualizar Gitea

1. Edita `docker-compose.yml` y cambia el número de versión de la línea `image`.
2. Descarga las imágenes nuevas y arranca:

```bash
sudo docker compose pull
sudo docker compose up -d
```

Antes de actualizar, haz una copia de seguridad (apartado 14).

---

## 13. Empezar de cero

Si has cometido un error en el asistente o quieres cambiar de base de datos **antes de usar Gitea**, puedes borrarlo todo:

```bash
cd ~/gitea
sudo docker compose down
sudo rm -rf data config postgres
mkdir -p data config
```

!!! warning "Esto borra todo"
    Se pierden los repositorios, usuarios y configuración de Gitea. Hazlo solo si no tienes nada que conservar. Después cambia el `docker-compose.yml` si quieres otra base de datos y vuelve al apartado 8.

---

## 14. Copia de seguridad básica

Con Gitea parado, copiar las carpetas es una copia de seguridad completa:

```bash
cd ~
sudo docker compose -f ~/gitea/docker-compose.yml stop
sudo tar czf gitea-copia.tar.gz gitea
sudo docker compose -f ~/gitea/docker-compose.yml start
```

Con PostgreSQL, además, puedes volcar la base de datos a un archivo de texto:

```bash
cd ~/gitea
sudo docker compose exec db pg_dump -U gitea gitea > copia-gitea.sql
```

---

## 15. Errores comunes y cómo resolverlos

| Qué ves | Causa probable | Qué hacer |
|---|---|---|
| `permission denied ... docker.sock` | Tu usuario no está en el grupo `docker` | Usa `sudo` delante, o añade tu usuario al grupo (apartado 4) |
| `hello-world` se queda a medias y aparece `^C` | Pulsaste `Ctrl + C` durante la descarga | Repite `sudo docker run hello-world` y espera |
| `Permission denied` / `not writable` en los registros | Las carpetas `data` o `config` no pertenecen al usuario 1000 | `sudo chown 1000:1000 config data` y `sudo docker compose up -d` |
| El asistente muestra `localhost:3306` con PostgreSQL | Estás usando el archivo de SQLite3 y has cambiado el tipo a mano | Usa el archivo de la opción B y empieza de cero (apartado 13) |
| El asistente no conecta con la base de datos | PostgreSQL aún está arrancando, o el servidor no es `db:5432` | Espera medio minuto, mira `docker compose ps` y revisa el campo Servidor |
| `address already in use` en el puerto 3000 | Otro programa ya usa ese puerto (otro Gitea, por ejemplo) | `sudo ss -tlnp \| grep 3000` para ver cuál es |
| El usuario `admin` es rechazado | Es un nombre reservado de Gitea | Elige otro nombre |
| Los contenedores no se ven con `docker ps` | Estás en otra máquina o sin `sudo` | `sudo docker ps` |
| La página no carga | Gitea está parado o arrancando | `sudo docker compose ps` y `sudo docker compose logs --tail 30` |

### Dónde mirar cuando algo falla

```bash
sudo docker compose ps                    # ¿están arriba los contenedores?
sudo docker compose logs --tail 30        # ¿qué ha escrito Gitea?
sudo ss -tlnp | grep 3000                 # ¿quién usa el puerto 3000?
cat docker-compose.yml                    # ¿está bien el archivo?
```

---

## 16. Chuleta rápida

```bash
# Instalar Docker y Compose (versión corta)
sudo apt update
sudo apt install docker.io docker-compose-v2
sudo docker run hello-world

# Crear el proyecto
mkdir -p ~/gitea/{data,config}
cd ~/gitea
id -u                                  # si no es 1000: sudo chown 1000:1000 config data

# Crear docker-compose.yml (opción A o B del apartado 7)
nano docker-compose.yml

# Arrancar
sudo docker compose up -d
sudo docker compose ps

# Abrir en el navegador y completar el asistente
# http://127.0.0.1:3000

# Día a día
sudo docker compose stop
sudo docker compose start
sudo docker compose logs --tail 30
sudo docker compose down
```

---

## 17. Resumen del flujo completo

```
docker-compose.yml  →  docker compose up -d  →  contenedor(es) en marcha
                                                        │
Navegador → http://127.0.0.1:3000 → Gitea ──────────────┤
                                       │                │
                                       ├─ SQLite3: archivo dentro de data/
                                       └─ PostgreSQL: contenedor db → carpeta postgres/
```

Una cosa que recordar: **los datos viven en carpetas de tu máquina** (`data`, `config` y `postgres`), no dentro de la caja. Por eso puedes parar, borrar y volver a crear los contenedores sin perder nada.

---

## Recursos

- [Gitea: instalación con Docker (rootless)](https://docs.gitea.com/installation/install-with-docker-rootless/)
- [Gitea: instalación con Docker](https://docs.gitea.com/installation/install-with-docker/)
- [Gitea: preparación de la base de datos](https://docs.gitea.com/installation/database-prep/)
- [Docker Engine en Ubuntu](https://docs.docker.com/engine/install/ubuntu/)
