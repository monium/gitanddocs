# Uv y Zensical

Guía para instalar `uv`, configurar **Zensical**, levantar el servidor de desarrollo y aprovechar la recarga en tiempo real.

---

## Paso 1: Instalar `uv`

`uv` es un gestor de paquetes y entornos de Python, abre tu terminal y ejecuta:

```
curl -sSf https://astral.sh/uv/install.sh | sh
```

> **Nota:** Tras completar la instalación, cierra y vuelve a abrir tu terminal para asegurarte de que el comando `uv` está disponible.

**Comprobar la ruta del ejecutable:**
   
```
which uv
```
*(Debería apuntar habitualmente a `~/.cargo/bin/uv` o `~/.local/bin/uv`)*

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
<<<<<<< HEAD

<br>
<br>

# Desplegar site en un contenedor con NGINX

### Opción 1: Sitio estático (HTML/CSS/JS)

**Dockerfile**
```dockerfile
FROM nginx:alpine

# Copia tus archivos estáticos a la carpeta que sirve NGINX
COPY ./dist /usr/share/nginx/html

# Copia tu configuración personalizada de NGINX
COPY nginx.conf /etc/nginx/conf.d/default.conf

EXPOSE 80
```

**nginx.conf**
```nginx
server {
    listen 80;
    server_name _;

    root /usr/share/nginx/html;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }

    # Cache de assets estáticos
    location ~* \.(js|css|png|jpg|jpeg|gif|svg|ico|woff2?)$ {
        expires 30d;
        add_header Cache-Control "public, immutable";
    }
}
```

**Construir y ejecutar**
```bash
docker build -t mi-sitio .
docker run -d -p 80:80 --name mi-sitio-container mi-sitio
```

### Opción 2: WordPress (NGINX + PHP-FPM + MySQL)

**docker-compose.yml**
```yaml
version: "3.8"

services:
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf
      - wp_data:/var/www/html
    depends_on:
      - wordpress

  wordpress:
    image: wordpress:php8.2-fpm-alpine
    environment:
      WORDPRESS_DB_HOST: db
      WORDPRESS_DB_NAME: wordpress
      WORDPRESS_DB_USER: wpuser
      WORDPRESS_DB_PASSWORD: wppass
    volumes:
      - wp_data:/var/www/html

  db:
    image: mysql:8.0
    environment:
      MYSQL_DATABASE: wordpress
      MYSQL_USER: wpuser
      MYSQL_PASSWORD: wppass
      MYSQL_ROOT_PASSWORD: rootpass
    volumes:
      - db_data:/var/lib/mysql

volumes:
  wp_data:
  db_data:
```

**Ejecutar**
```bash
docker compose up -d
```

### Puntos clave

- **HTTPS**: en producción, añade un proxy con Certbot/Let's Encrypt delante (o usa Traefik/Caddy para TLS automático).
- **Persistencia**: usa volúmenes para no perder datos al reconstruir el contenedor.
- **Logs**: monta `/var/log/nginx` como volumen si quieres revisarlos fuera del contenedor.
- **Reinicio automático**: añade `restart: unless-stopped` en el compose.
=======
>>>>>>> parent of 926b267 (Como usar UV sin recargar el terminal)
