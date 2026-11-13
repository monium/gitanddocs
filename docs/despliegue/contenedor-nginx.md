---
icon: simple/nginx
---

# Desplegar el contenedor

## Construir la imagen

```bash
docker build -t mi-sitio .
```

## Ejecutar el contenedor

```bash
docker run -d -p 80:80 --name mi-sitio-container mi-sitio
```

- `-d`: lo ejecuta en segundo plano.
- `-p 80:80`: expone el puerto 80 del contenedor en el puerto 80 de tu máquina.
- `--name`: nombre del contenedor.
- `mi-sitio-container`: nombre que le pones al contenedor
- `mi-sitio`: la imagen que usa como base

Después puedes abrir `http://localhost` en el navegador para comprobar que funciona.

### Comandos útiles

```bash
# Ver contenedores en ejecución
docker ps

# Ver los logs
docker logs mi-sitio-container

# Parar y arrancar el contenedor
docker stop mi-sitio-container
docker start mi-sitio-container

# Eliminar el contenedor
docker rm -f mi-sitio-container
```

## Actualizar el sitio tras hacer cambios

```bash
docker build -t mi-sitio .
docker rm -f mi-sitio-container
docker run -d -p 80:80 --name mi-sitio-container mi-sitio
```

### Con docker-compose (opcional)

**docker-compose.yml**

```yaml
services:
  web:
    build: .
    ports:
      - "80:80"
    restart: unless-stopped
```

```bash
docker compose up -d --build
```