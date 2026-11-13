# Instalación de Gitea con Docker

Primero crearía una carpeta para el servicio:

```bash
mkdir gitea
cd gitea
```
A continuación, crearía un archivo `docker-compose.yml` con la configuración de Gitea. Un ejemplo básico sería:

```yaml
services:
  server:
    image: docker.gitea.com/gitea:28.1.0
    container_name: gitea
    environment:
      - USER_UID=1000
      - USER_GID=1000
    restart: always
    volumes:
      - ./gitea:/data
      - /etc/timezone:/etc/timezone:ro
      - /etc/localtime:/etc/localtime:ro
    ports:
      - "3000:3000"
      - "222:22"
```

Esta configuración utiliza la imagen oficial de Gitea y guarda los datos en ./gitea, por lo que los repositorios y la configuración no se perderán si el contenedor se reinicia. También se publican los puertos 3000, utilizado para acceder a la interfaz web, y 222, utilizado para las conexiones Git mediante SSH.

Después se inicia el servicio con:
```bash
docker compose up -d
```

Este comando descarga la imagen de Gitea si todavía no está disponible y crea el contenedor en segundo plano. Para comprobar que se ha iniciado correctamente:

```bash
docker compose ps
```

También se pueden consultar los registros con:

```bash
docker compose logs
```

La documentación oficial indica estos comandos para iniciar, comprobar y consultar los registros del servicio.

# Acceso a Gitea
Una vez iniciado el contenedor, se puede acceder desde un navegador mediante:
http://IP_DEL_SERVIDOR:3000

Por ejemplo, si el servidor tuviera la IP 192.168.1.100:
http://192.168.1.100:3000

Desde la pantalla inicial se completa la configuración de Gitea y se crea la primera cuenta, que puede utilizarse como administrador.

# Uso en una empresa
Una vez instalado, el administrador puede crear usuarios para los empleados y organizaciones o repositorios corporativos. Cada empleado podrá acceder con su propia cuenta y trabajar con los repositorios que tenga autorizados.

Por ejemplo, se podría crear una organización llamada Empresa y dentro de ella tener:

Empresa
├── proyecto-web
├── aplicacion-java
├── documentacion
└── tienda-online

Los empleados podrían clonar los repositorios mediante Git, realizar cambios y subirlos al servidor Gitea. Además, Gitea permite gestionar permisos, ramas protegidas, incidencias y pull requests, por lo que puede utilizarse como plataforma centralizada de desarrollo.

# Base de datos

Para una instalación sencilla se puede utilizar SQLite, pero para un entorno profesional sería más recomendable utilizar una base de datos como PostgreSQL. La propia documentación de Gitea proporciona una configuración mediante Docker Compose con PostgreSQL y mantiene los datos mediante volúmenes.

También sería recomendable configurar HTTPS, copias de seguridad y contraseñas seguras antes de utilizar el servicio en producción.

# Conclusión

En resumen, instalaría Gitea en un servidor de la empresa mediante Docker Compose. Los empleados accederían a Gitea desde sus ordenadores utilizando la IP o dominio del servidor, cada uno con su propia cuenta y permisos. De esta forma, todos los repositorios corporativos estarían centralizados en un único servidor privado y se podría controlar quién puede acceder o modificar cada proyecto.