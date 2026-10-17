---
icon: lucide/key-round
---

# Crear una clave SSH y comprobar que funciona

Guía paso a paso para **generar un par de claves SSH**, entender qué archivos se crean, **dónde se guardan** y **cómo probar** que la autenticación funciona con `ssh -T`.

Al terminar sabrás:

- Crear una clave SSH con `ssh-keygen`.
- Distinguir la **clave pública** (`.pub`) de la **clave privada**.
- Saber **dónde viven** las claves por defecto.
- **Comprobar** si ya tienes una clave creada.
- **Probar** la clave contra un servicio (`ssh -T git@host`).
- **Usar otra clave** que esté guardada en otra ruta.
- Qué es y para qué sirve el archivo `~/.ssh/known_hosts`.

---

## 1. ¿Qué es una clave SSH?

SSH usa **criptografía de clave pública**: se generan dos archivos que van en pareja.

- Una **clave privada** que se queda **solo en tu ordenador**. Nunca se comparte.
- Una **clave pública** que se copia al servidor (o se sube a GitHub/Gitea) para que este te reconozca.

Cuando te conectas, el servidor comprueba que la clave pública que tiene registrada encaja con tu clave privada sin que la privada salga nunca de tu máquina. Es más seguro que una contraseña porque **nadie puede robar lo que nunca se envía**.

---

## 2. Comprobar si ya tienes una clave SSH creada

Antes de crear una clave nueva, mira si ya existe alguna:

```bash
ls -la ~/.ssh
```

Busca archivos como `id_ed25519` (privada) e `id_ed25519.pub` (pública). También sirve:

```bash
ls -la ~/.ssh/*.pub
```

Si ves algo como `id_ed25519.pub` o `id_rsa.pub`, **ya tienes una clave** y no necesitas crear otra (puedes reutilizarla). Si la carpeta no existe o está vacía, continúa con el siguiente paso.

!!! tip "Comando rápido"
    ```bash
    test -f ~/.ssh/id_ed25519.pub && echo "Ya existe una clave" || echo "No hay clave, hay que crearla"
    ```

---

## 3. Generar un par de claves SSH

Ejecuta `ssh-keygen`. El algoritmo recomendado hoy es **ED25519** (moderno, corto y seguro):

```bash
ssh-keygen -t ed25519 -C "tu_correo@ejemplo.com"
```

- `-t ed25519`: tipo de clave.
- `-C "tu_correo@ejemplo.com"`: un comentario para identificarla (normalmente tu email). Es opcional, pero ayuda a saber de quién es.

El comando te hará **tres preguntas**:

1. **Dónde guardar la clave** → pulsa `Enter` para aceptar la ruta por defecto.
2. **Passphrase** (frase de contraseña) → opcional, pero recomendada. Se te solicitará cada vez que desees hacer uso de la clave.
3. **Repetir la passphrase**.

Verás una salida parecida a esta:

```text
Generating public/private ed25519 key pair.
Enter file in which to save the key (/home/usuario/.ssh/id_ed25519):
Enter passphrase (empty for no passphrase):
Your identification has been saved in /home/usuario/.ssh/id_ed25519
Your public key has been saved in /home/usuario/.ssh/id_ed25519.pub
The key fingerprint is:
SHA256:AbCdEf... tu_correo@ejemplo.com
```

!!! note "¿Ed25519 o RSA?"
    Usa `ed25519` salvo que necesites compatibilidad con equipos muy antiguos. Si aún así necesitas RSA, usa `ssh-keygen -t rsa -b 4096`.

---

## 4. Diferencia entre la clave pública (`.pub`) y la clave privada

`ssh-keygen` crea **dos archivos** en `~/.ssh`:

| Archivo | Nombre | ¿Qué es? | ¿Se comparte? |
|---|---|---|---|
| `id_ed25519` | **Clave privada** | Tu secreto. Firma las conexiones. | **Nunca**. Se queda en tu equipo. |
| `id_ed25519.pub` | **Clave pública** | Se copia al servidor o a GitHub/Gitea. | Sí, es pública. |

La clave pública **empieza siempre por el tipo de clave** y es **una sola línea larga**:

```text
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI... tu_correo@ejemplo.com
```

La clave privada, en cambio, es un bloque de varias líneas delimitado por cabeceras:

```text
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAA...
-----END OPENSSH PRIVATE KEY-----
```

!!! warning "Nunca compartas la clave privada"
    Si pegas por error tu clave privada en un chat, un correo o un repositorio, considérala comprometida: bórrala y genera un par nuevo.

!!! tip "Regla mental sencilla"
    La que **termina en `.pub`** es la que se **copia/pega** en los sitios. La que **no** termina en `.pub` es tu **secreto**, y solo se usa desde tu ordenador.

---

## 5. ¿Dónde se guardan las claves SSH por defecto?

Por defecto, todo vive en la carpeta **`~/.ssh`** dentro de tu usuario:

- En **Linux/macOS**: `/home/usuario/.ssh` o `/Users/usuario/.ssh`.
- En **Windows**: `C:\Users\usuario\.ssh` (con OpenSSH).

Los nombres más habituales son:

| Ruta (por defecto) | Descripción |
|---|---|
| `~/.ssh/id_ed25519` | Clave privada (ED25519) |
| `~/.ssh/id_ed25519.pub` | Clave pública (ED25519) |
| `~/.ssh/id_rsa` / `id_rsa.pub` | Par RSA (esquema antiguo) |
| `~/.ssh/authorized_keys` | Claves públicas autorizadas (**en el servidor**) |
| `~/.ssh/known_hosts` | Huellas de servidores ya visitados |
| `~/.ssh/config` | Configuración de conexiones (alias, qué clave usar, etc.) |

Si no indicas ninguna clave al conectarte, SSH usará **automáticamente** los nombres por defecto (`id_ed25519`, `id_rsa`...) de esa carpeta.

Vista rápida de la estructura:

```text
~/.ssh/
├── id_ed25519          ← clave privada (secreta)
├── id_ed25519.pub      ← clave pública (compartible)
├── known_hosts         ← servidores conocidos
├── config              ← opciones de conexión
└── authorized_keys     ← (esto vive en el SERVIDOR, no en tu equipo)
```

---

## 6. Probar que la clave funciona con `ssh -T`

La opción **`-T`** le dice a SSH que **no abra una sesión interactiva** (sin shell): solo comprueba si la autenticación funciona y sale. Es la forma estándar de probar la clave contra servicios Git como **GitHub**, **GitLab** o **Gitea**.

!!! warning "Contraseña vs `-T`"
    `ssh -T` sirve para probar autenticación contra **servicios Git** (`git@github.com`, `git@gitea.midominio.com`...). Para entrar a una **máquina normal** no se usa `-T`; ahí abres sesión con `ssh usuario@host`.

### Probar contra GitHub

```bash
ssh -T git@github.com
```

Si la clave está bien configurada, responde algo como:

```text
Hi tu_usuario! You've successfully authenticated, but GitHub does not provide shell access.
```

Ese mensaje **significa que funciona**: te ha reconocido.

### Probar contra GitLab o Gitea

```bash
# GitLab
ssh -T git@gitlab.com

# Gitea (usa el puerto SSH que tenga configurado, a menudo 22 o 2222)
ssh -T git@gitea.midominio.com
ssh -p 2222 -T git@gitea.midominio.com
```

### Si falla

- **`Permission denied (publickey)`** → tu clave pública no está registrada en el servicio. Cópiala en **GitHub → Settings → SSH and GPG keys** o en **Gitea → Configuración → Claves SSH**.
- **`Host key verification failed`** → el servidor no está en `known_hosts` (ver sección 7) o cambió su huella.

Puedes ver qué claves está ofreciendo tu cliente SSH con:

```bash
ssh -vT git@github.com
```

La opción `-v` (verbose) muestra qué archivos de clave está usando por el camino; muy útil para depurar.

---

## 7. Qué es `~/.ssh/known_hosts`

Cuando te conectas por primera vez a un servidor, este te envía su **huella** (fingerprint). SSH te pregunta si confías en él:

```text
The authenticity of host 'github.com (140.82.121.4)' can't be established.
ED25519 key fingerprint is SHA256:xxxx...
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```

Al responder **`yes`**, SSH guarda esa huella en **`~/.ssh/known_hosts`**. 

**Para qué sirve:** en las siguientes conexiones SSH compara la huella que recibe con la guardada. Si **coincide**, no te vuelve a preguntar. Si **no coincide**, te avisa con una advertencia de posible **ataque de tipo "man in the middle"** y bloquea la conexión. Es una defensa contra servidores falsos que se hagan pasar por la máquina de confianza.

Es un archivo de texto plano: una huella por línea (a veces con el host en claro, a veces "hasheado" con `HashKnownHosts yes`).

!!! note "Gestionar servidores conocidos"
    ```bash
    # Ver los servidores de los que te fías
    cat ~/.ssh/known_hosts

    # Eliminar la entrada de un host concreto (p. ej. si reinstalaste el servidor y cambió su huella)
    ssh-keygen -R github.com
    ```

`ssh-keygen -R host` es la forma correcta de borrar una línea de `known_hosts`: la edición a mano puede dejar el archivo inconsistente.

---

## 8. Usar otra clave que está en otro lugar

Por defecto SSH usa las claves de `~/.ssh`. Si tu clave está en **otra ruta**, tienes varias formas de indicárselo.

### Opción A — Con `-i` en cada comando (rápida)

```bash
ssh -i /ruta/a/mi_clave usuario@host
ssh -T -i /ruta/a/mi_clave git@github.com
```

- `-i`, `--identity-file` apunta a la **clave privada** (la que **no** tiene `.pub`).

Recuerda copiar también la parte pública (`/ruta/a/mi_clave.pub`) al servicio donde quieras autenticarte.

### Opción B — Con `~/.ssh/config` (permanente, recomendado)

Si la usas a menudo, define un alias en `~/.ssh/config`:

```sshconfig
Host github-personal
    HostName github.com
    User git
    IdentityFile ~/.ssh/mis_claves/id_ed25519_personal
    IdentitiesOnly yes
```

A partir de ahí:

```bash
ssh -T github-personal
git clone github-personal:usuario/repo.git
```

- `IdentityFile`: ruta a la clave privada a usar.
- `IdentitiesOnly yes`: obliga a usar **solo** esa clave (evita que SSH pruebe otras y el servidor rechace por "too many authentication failures").

### Opción C — Cargar la clave en el agente SSH

El agente SSH recuerda claves desbloqueadas durante la sesión:

```bash
# Arranca el agente (si no está activo)
eval "$(ssh-agent -s)"

# Añade tu clave (te pedirá la passphrase una sola vez)
ssh-add /ruta/a/mi_clave

# Comprueba qué claves tiene cargadas
ssh-add -l
```

!!! tip "Permisos correctos"
    `ssh` es estricto con los permisos. La clave privada no debe ser legible por otros:

    ```bash
    chmod 700 ~/.ssh
    chmod 600 ~/.ssh/id_ed25519
    chmod 644 ~/.ssh/id_ed25519.pub
    ```

---

## 9. Ver la clave pública (para copiarla)

Para **copiar y pegar** la parte pública donde haga falta:

```bash
cat ~/.ssh/id_ed25519.pub
```

En **macOS** puedes copiarla directamente al portapapeles:

```bash
pbcopy < ~/.ssh/id_ed25519.pub
```

También es posible:
```bash
cat ~/.ssh/id_ed25519.pub | pbcopy
```

Y para comprobar la huella de una clave concreta:

```bash
ssh-keygen -lf ~/.ssh/id_ed25519.pub
```

---

## Recursos adicionales

- [Documentación oficial de OpenSSH](https://www.openssh.com/manual.html)
- [Generar una clave SSH en GitHub](https://docs.github.com/es/authentication/connecting-to-github-with-ssh/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent)
- [Claves SSH en GitLab](https://docs.gitlab.com/ee/user/ssh.html)
- [Guía relacionada: conectarse por SSH a una máquina Debian (desde cero)](connect-to-machine.md)
