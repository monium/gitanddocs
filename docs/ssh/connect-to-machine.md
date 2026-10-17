---
icon: lucide/terminal
---

# Conectarse por SSH a una máquina Debian (desde cero)

Tenemos una **máquina base con Debian** y queremos entrar en ella de forma **remota** desde otro equipo. Para eso está **SSH**, pero SSH no funciona «solo»: en esa máquina tiene que estar instalado y en marcha el **servidor SSH** (`sshd`).

Si compras una **VPS** o un servidor en la nube, normalmente ya te lo entregan con el servidor SSH instalado: te dan la **IP**, un **usuario** y una contraseña (o una clave) y te conectas desde el primer minuto. En esta guía vamos a hacerlo **desde cero** nosotros mismos, sobre una máquina Debian que levantamos con **Docker** a modo de ejemplo, para ver **cómo se instalan los paquetes y cómo se configura** el servicio. Al terminar sabrás:

- Levantar una máquina Debian de ejemplo con Docker.
- Instalar y arrancar el servidor SSH a mano.
- Crear un usuario y hacer la primera conexión con contraseña.
- Configurar la **autenticación con clave pública**.
- **Deshabilitar** la contraseña y comprobar que ya no sirve.

!!! note "Lo que aprendas aquí vale para una VPS real"
    Los comandos *dentro de la máquina* son los mismos que usarías en un servidor Debian real. Solo cambian dos cosas: **cómo te conectas** (aquí a `127.0.0.1`, puerto `2222`; en una VPS, a su IP, puerto `22`) y que en el contenedor **no hay systemd**, así que los servicios se manejan con `service` en lugar de `systemctl`.

---

## 1. La «máquina remota» de ejemplo: un contenedor Debian

Un contenedor de Docker es como una máquina Linux pequeña y completa: se instalan paquetes con `apt`, se crean usuarios, etc. Nos servirá de **máquina remota** sin tocar tu sistema.

Necesitas:

- **Docker** instalado y funcionando en tu equipo. Si no lo tienes, mira el apartado 4 de [Instalar Gitea con Docker](../gitea/Install-Gitea-with-docker.md).
- Un **cliente SSH**: Linux y macOS ya lo traen; en Windows se instala como función opcional *OpenSSH Client*.

Crea y arranca la máquina:

```bash
docker run -d --name maquina-debian -p 2222:22 debian:trixie sleep infinity
```

| Parte | Qué significa |
|---|---|
| `-d` | Ejecutar en segundo plano (no bloquea tu terminal) |
| `--name maquina-debian` | Un nombre para referirte a ella después |
| `-p 2222:22` | Publica el puerto `22` **de la máquina** en el `2222` **de tu equipo** |
| `debian:trixie` | La imagen oficial de **Debian 13** |
| `sleep infinity` | Un proceso que no hace nada pero mantiene la máquina encendida (un contenedor se apaga cuando termina su proceso principal) |

!!! tip "El `-p 2222:22` solo lo usamos aquí"
    Dentro de la máquina, SSH escuchará en el puerto estándar `22`. En un servidor real te conectas a ese puerto directamente, sin indicar nada.

Entra en la **consola** de la máquina (es como sentarte delante de ella):

```bash
docker exec -it maquina-debian bash
```

Comprueba qué máquina es:

```bash
cat /etc/os-release
```

Verás `Debian GNU/Linux 13 (trixie)`. Ya estás «dentro» y eres `root`.

!!! tip "Esta consola es tu acceso de rescate"
    Deja esta terminal abierta. Es el equivalente a tener acceso físico a la máquina: aunque el SSH deje de funcionar, desde aquí siempre podrás arreglarlo.

---

## 2. Instalar el servidor SSH

Dentro de la consola, actualiza el catálogo de paquetes e instala el servidor:

```bash
apt update
apt install openssh-server sudo
```

| Comando | Qué hace |
|---|---|
| `apt update` | Actualiza la lista de paquetes disponibles (no instala nada) |
| `apt install openssh-server sudo` | Instala el **servidor SSH** y `sudo` (para administrar la máquina después desde la sesión de `usuario`; en una máquina real `sudo` ya viene instalado) |

Pulsa `Enter` cuando pregunte `Do you want to continue?`.

!!! tip "¿Por qué se instalan tantos paquetes?"
    `apt` instala también las dependencias de `openssh-server` (entre ellas `systemd`, aunque en el contenedor no lo usemos). Es normal.

`openssh-server` deja en la máquina, entre otras cosas:

| Elemento | Para qué sirve |
|---|---|
| `/usr/sbin/sshd` | El programa servidor que atiende las conexiones |
| `/etc/ssh/sshd_config` | Su configuración (+ la carpeta `/etc/ssh/sshd_config.d/`) |
| `/etc/ssh/ssh_host_*` | Las claves propias de la máquina (su «DNI»). El cliente las usa para comprobar que hablas con la máquina correcta; se generan al instalar |
| Usuario `sshd` | Cuenta interna sin privilegios con la que el servicio se aísla por seguridad |

### Arrancar el servicio

```bash
service ssh start
service ssh status
```

`service ssh status` debe responder `sshd is running.`

!!! note "¿Por qué no arranca solo?"
    En una máquina Debian normal (con **systemd**), al instalar el paquete el servicio **arranca solo**. En el contenedor no hay systemd y, además, las imágenes de Docker llevan un pequeño script (`policy-rc.d`) que impide a `apt` arrancar servicios; por eso damos la orden a mano. Ese es el único cambio respecto a una máquina normal.

Equivalencias entre una máquina real y el contenedor (desde la consola ya eres `root`, así que `sudo` no hace falta ahí):

| Acción | Máquina real (systemd) | Contenedor (esta guía) |
|---|---|---|
| Arrancar | `sudo systemctl start ssh` | `service ssh start` |
| Ver estado | `sudo systemctl status ssh` | `service ssh status` |
| Recargar configuración | `sudo systemctl reload ssh` | `service ssh reload` |
| Parar | `sudo systemctl stop ssh` | `service ssh stop` |

---

## 3. Crear un usuario para conectarnos

No nos conectaremos como `root`: Debian lo bloquea por defecto (`PermitRootLogin prohibit-password`; root solo podría entrar con clave, nunca con contraseña) y usar un usuario normal es más seguro. Crea uno:

```bash
adduser usuario
```

Te pedirá:

1. Una **contraseña** (dos veces). Apúntala: la usarás en la primera conexión.
2. Datos como el nombre completo, que puedes dejar vacíos pulsando `Enter`.

Al final, confirma con `Y`. Después dale permiso de administración:

```bash
adduser usuario sudo
```

---

## 4. Primera conexión (con contraseña)

Abre **otra terminal** en tu equipo (deja la consola del contenedor abierta) y conéctate:

```bash
ssh -p 2222 usuario@127.0.0.1
```

La primera vez pasarán dos cosas:

1. Te preguntará si confías en la máquina (`Are you sure you want to continue connecting?`). Responde `yes`: es normal la primera vez; su huella se guarda en `~/.ssh/known_hosts` (se explica en [Crear una clave SSH](create-ssh-key.md)).
2. Te pedirá la **contraseña** que pusiste en `adduser`.

Si has entrado, comprueba dónde estás y sal:

```bash
whoami      # usuario
hostname    # el nombre del contenedor
exit        # volver a tu equipo
```

!!! note "En una VPS real"
    El comando sería `ssh usuario@IP_DE_LA_VPS` (sin `-p`: el puerto por defecto ya es el 22). Si no conecta, revisa que el cortafuegos del proveedor (el «grupo de seguridad») permita el puerto 22.

---

## 5. Autenticación con clave pública

Las contraseñas se pueden adivinar o robar. La alternativa es un **par de claves**: la privada se queda siempre en tu equipo (nunca se envía) y la pública se copia a la máquina.

### 5.1. Generar el par de claves (en tu equipo)

Si aún no tienes una, genérala siguiendo [Crear una clave SSH](create-ssh-key.md). Esta guía asume la ruta por defecto: `~/.ssh/id_ed25519` (privada) y `~/.ssh/id_ed25519.pub` (pública).

### 5.2. Copiar la clave pública a la máquina

```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub -p 2222 usuario@127.0.0.1
```

Te pedirá la contraseña **por última vez**: la clave pública acabará en el archivo `~/.ssh/authorized_keys` de la máquina. Si `ssh-copy-id` no está disponible (por ejemplo, en Windows), cópiala a mano:

```bash
cat ~/.ssh/id_ed25519.pub | ssh -p 2222 usuario@127.0.0.1 \
  "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

### 5.3. Probar la clave

```bash
ssh -i ~/.ssh/id_ed25519 -p 2222 usuario@127.0.0.1
```

Esta vez **no debe pedir contraseña**.

!!! tip "El argumento `-i`"
    `-i` indica qué clave privada usar. Si tu clave está en la ruta por defecto (`~/.ssh/id_ed25519`), puedes omitirlo: `ssh -p 2222 usuario@127.0.0.1`.

---

## 6. Deshabilitar el acceso por contraseña

Ahora que la clave funciona, quitamos la entrada con contraseña: aunque alguien la adivine, no podrá usarla. Este cambio se hace **en la máquina**.

!!! warning "No cierres la sesión SSH ni la consola"
    Mantén abiertas la sesión SSH actual y la consola del contenedor hasta terminar y probarlo todo en otra terminal. Si algo sale mal, desde cualquiera de las dos podrás revertirlo (y la consola funciona aunque el SSH quede roto).

### 6.1. Escribir la configuración

La forma más limpia es crear un **archivo adicional** en `/etc/ssh/sshd_config.d/`, que Debian incluye automáticamente:

```bash
sudo tee /etc/ssh/sshd_config.d/00-no-password.conf > /dev/null <<'EOF'
PubkeyAuthentication yes
PasswordAuthentication no
KbdInteractiveAuthentication no
ChallengeResponseAuthentication no
PermitEmptyPasswords no
EOF
```

| Directiva | Qué hace |
|---|---|
| `PubkeyAuthentication yes` | Permite entrar con clave pública (lo que queremos conservar) |
| `PasswordAuthentication no` | **Desactiva** la entrada con contraseña |
| `KbdInteractiveAuthentication no` | Desactiva la autenticación interactiva (preguntas y respuestas) |
| `ChallengeResponseAuthentication no` | Nombre antiguo de la anterior; se deja por compatibilidad |
| `PermitEmptyPasswords no` | Prohíbe las cuentas sin contraseña |

!!! warning "El nombre del archivo importa"
    En `sshd_config.d` los archivos se leen **en orden alfabético** y gana el **primero** que define una opción. Por eso usamos el prefijo `00-`: así se impone a cualquier otro archivo que ya exista (muchas VPS traen un `50-cloud-init.conf` que puede reactivar la contraseña). Mira lo que hay con `ls /etc/ssh/sshd_config.d/`.

!!! tip "Alternativa: editar el archivo principal"
    También puedes editar `/etc/ssh/sshd_config` a mano (con `sudo nano /etc/ssh/sshd_config`) y ajustar ahí esas mismas líneas. Si editas los dos sitios, recuerda que gana el **primer** valor leído.

### 6.2. Validar y recargar

Antes de recargar, comprueba que no haya errores (si no muestra nada, está bien):

```bash
sudo sshd -t
```

Recarga el servicio (no corta las conexiones abiertas):

```bash
sudo service ssh reload
# En una máquina real:
# sudo systemctl reload ssh
```

### 6.3. Comprobar la configuración efectiva

```bash
sudo sshd -T | grep -E 'passwordauthentication|pubkeyauthentication|kbdinteractiveauthentication|permitemptypasswords'
```

Debe aparecer `passwordauthentication no` y `pubkeyauthentication yes`. Si la contraseña sigue en `yes`, revisa el orden de los archivos de `sshd_config.d` (apartado anterior).

---

## 7. Probar la nueva conexión

**Sin cerrar** la sesión que tienes abierta, abre otra terminal y entra con la clave:

```bash
ssh -i ~/.ssh/id_ed25519 -p 2222 usuario@127.0.0.1
```

Debe entrar directo, sin pedir contraseña. Y ahora la prueba definitiva: forzar un intento con contraseña.

```bash
ssh -o PubkeyAuthentication=no -p 2222 usuario@127.0.0.1
```

Debe fallar con:

```text
usuario@127.0.0.1: Permission denied (publickey).
```

Ese error es la **buena noticia**: la máquina ya no acepta contraseñas, solo claves. Si en su lugar te pidiera la contraseña, la configuración no se aplicó (revisa el apartado 6).

---

## 8. Apagar y borrar la máquina de ejemplo

Cuando termines de practicar:

```bash
docker stop maquina-debian   # apagarla (puedes volver a encenderla con docker start)
docker rm -f maquina-debian  # destruirla: se pierde todo lo instalado dentro
```

Si vas a crearla de nuevo, borra la huella antigua (la máquina nueva tendrá claves de host nuevas):

```bash
ssh-keygen -R "[127.0.0.1]:2222"
```

!!! tip "Rehacerla entera en un momento"
    ```bash
    docker run -d --name maquina-debian -p 2222:22 debian:trixie sleep infinity
    docker exec -it maquina-debian bash
    # (dentro de la consola)
    apt update && apt install -y openssh-server sudo
    service ssh start
    adduser usuario
    adduser usuario sudo
    # sal de la consola y copia tu clave:
    ssh-copy-id -i ~/.ssh/id_ed25519.pub -p 2222 usuario@127.0.0.1
    ```

---

## 9. Solución de problemas

### En el contenedor

- **`Connection refused` al conectar**: el servidor SSH no está arrancado o el puerto no está publicado.
  ```bash
  service ssh status          # dentro de la máquina: ¿dice "sshd is running."?
  docker port maquina-debian  # en tu equipo: debe mostrar 22/tcp -> 0.0.0.0:2222
  ```
- **`REMOTE HOST IDENTIFICATION HAS CHANGED!`**: borraste y recreaste el contenedor, así que la máquina tiene claves nuevas. Borra la huella antigua y vuelve a conectar:
  ```bash
  ssh-keygen -R "[127.0.0.1]:2222"
  ```
- **`sshd: no hostkeys available -- exiting`**: faltan las claves propias de la máquina. Genéralas y arranca:
  ```bash
  sudo ssh-keygen -A
  sudo service ssh start
  ```
- **Ver los registros del servidor**: en el contenedor no hay journal, así que páralo y arráncalo en primer plano para verlos en directo:
  ```bash
  sudo service ssh stop
  sudo /usr/sbin/sshd -D -e     # Ctrl + C para pararlo; luego: sudo service ssh start
  ```

### En una máquina real

- Revisa los logs del servidor:
  ```bash
  sudo journalctl -u ssh -e
  # o
  sudo journalctl -u sshd -e
  ```

### En general

- **`Permission denied (publickey)`** con la clave: no está registrada o los permisos están mal. En la máquina:
  ```bash
  chmod 700 ~/.ssh
  chmod 600 ~/.ssh/authorized_keys
  ```
  Comprueba además que la clave pública correcta está dentro de `~/.ssh/authorized_keys`. Desde tu equipo, `ssh -v usuario@host` muestra qué claves está probando el cliente.
- **La contraseña sigue funcionando tras deshabilitarla**: revisa la precedencia de `sshd_config.d` (gana el primer archivo en orden alfabético) y vuelve a comprobar con `sudo sshd -T | grep passwordauthentication`.
- **`sshd -t` muestra errores**: corrige el archivo antes de recargar; una errata deja el servidor sin arrancar.
- **El puerto 2222 está ocupado en tu equipo**: usa otro, por ejemplo `-p 2223:22`, y conecta con `-p 2223`.

---

## Recursos adicionales

- [Documentación oficial de OpenSSH](https://www.openssh.com/manual.html)
- [Página sobre SSH en la wiki de Debian](https://wiki.debian.org/SSH)
- `man sshd_config` y `man sshd` (en una máquina real; en el contenedor las páginas `man` no están instaladas)
- [Guía relacionada: crear una clave SSH](create-ssh-key.md)
