# Acceso SSH solo con clave (sin contraseña) en un servidor

## ¿Qué vamos a conseguir?

Por defecto, un servidor permite entrar por SSH con **usuario y contraseña**. Esto es peligroso ya que podemos recibir ataques de *fuerza bruta*.

En esta guía haremos que **solo pueda entrar quien tenga una clave SSH**, y desactivaremos el acceso con contraseña.

"¿Qué es una clave SSH?"
Es un par de archivos que funcionan como una **llave y una cerradura**:

- **Clave privada** (la llave): se queda en tu ordenador. **Nunca se comparte.**
- **Clave pública** (la cerradura): se copia al servidor. Se puede compartir sin problema.

Cuando te conectas, el servidor comprueba que tu llave encaja en su cerradura. No hace falta escribir ninguna contraseña.

---

## Antes de empezar

Necesitas:

- Un servidor con acceso SSH funcionando.
- La **IP** del servidor y tu **usuario**.
- Una terminal en tu ordenador.

En los ejemplos usaremos:

| Dato | Ejemplo |
|------|---------|
| Usuario | `usuario` |
| IP del servidor | `192.168.0.1` |

Sustitúyelos por los tuyos.

**No cierres la sesión SSH que tengas abierta** hasta terminar toda la guía y comprobar que todo funciona. Si te equivocas, esa sesión te permitirá arreglarlo. Si la cierras, podrías quedarte sin acceso a tu servidor.

---

## Paso 1: Crear la clave SSH (en tu ordenador)

Abre una terminal **en tu ordenador** (no en el servidor) y ejecuta:

```bash
ssh-keygen -t ed25519 -C "mi-clave"
```

Te hará tres preguntas:

1. **Dónde guardar la clave**: pulsa `Enter` para aceptar la ruta por defecto.
2. **Passphrase (frase de contraseña)**: es recomendable poner una. Protege tu clave si alguien tiene acceso físico a tu ordenador.
3. **Repetir la passphrase**.

Se habrán creado dos archivos en la carpeta `~/.ssh/`:

| Archivo | Qué es |
|---------|--------|
| `id_ed25519` | Clave **privada**. No la compartas nunca. |
| `id_ed25519.pub` | Clave **pública**. Esta es la que irá al servidor. |

---

## Paso 2: Copiar la clave pública al servidor

### Opción A: Linux y macOS

```bash
ssh-copy-id usuario@192.168.0.1
```

Te pedirá la contraseña del servidor por última vez. Listo.

### Opción B: Windows (PowerShell)

```powershell
type $env:USERPROFILE\.ssh\id_ed25519.pub | ssh usuario@192.168.0.1 "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
```

### Opción C: Manual

1. Muestra tu clave pública en tu ordenador y cópiala entera:

    ```bash
    cat ~/.ssh/id_ed25519.pub
    ```

2. Conéctate al servidor con contraseña y ejecuta:

    ```bash
    mkdir -p ~/.ssh
    vim ~/.ssh/authorized_keys
    ```

3. Pega la clave en una línea nueva y guardamos el fichero.

4. Ajusta los permisos:

    ```bash
    chmod 700 ~/.ssh
    chmod 600 ~/.ssh/authorized_keys
    ```

---

## Paso 3: Probar que la clave funciona

Abre una **nueva terminal** (deja la anterior abierta) y conéctate:

```bash
ssh usuario@192.168.0.1
```

- Si has puesto passphrase, te pedirá **la passphrase de la clave** (no la contraseña del servidor).
- Si entras correctamente, la clave funciona. Puedes continuar.

Si te sigue pidiendo la contraseña del servidor o te da error, **no continúes**. Revisa el Paso 2 (normalmente es un problema de permisos o de que la clave se pegó mal).

---

## Paso 4: Desactivar el acceso con contraseña (en el servidor)

Ya conectado al servidor, edita la configuración del servicio SSH:

```bash
sudo vim /etc/ssh/sshd_config
```

Busca (o añade al final) estas líneas y déjalas así:

```text
PasswordAuthentication no
KbdInteractiveAuthentication no
PubkeyAuthentication yes
```

Qué hace cada una:

| Opción | Significado |
|--------|-------------|
| `PasswordAuthentication no` | Prohíbe entrar con contraseña. |
| `KbdInteractiveAuthentication no` | Cierra otra vía por la que también se podría pedir contraseña. |
| `PubkeyAuthentication yes` | Permite entrar con clave SSH. |

"Opcional: bloquear el acceso directo de root"
Para más seguridad, añade también:

```text
PermitRootLogin no
```

Hazlo solo si tienes otro usuario con permisos `sudo`, o te quedarás sin acceso administrador.

Guardamos el fichero.

### Revisa archivos que pueden "pisar" tu configuración

En muchos servidores (especialmente Ubuntu), hay archivos extra que **tienen prioridad** sobre `sshd_config` y pueden reactivar la contraseña sin que te des cuenta. Compruébalo:

```bash
sudo grep -r "PasswordAuthentication" /etc/ssh/sshd_config.d/
```

Si aparece algo como `PasswordAuthentication yes` en algún archivo (por ejemplo `50-cloud-init.conf`), edítalo y cámbialo a `no`:

```bash
sudo vim /etc/ssh/sshd_config.d/50-cloud-init.conf
```

---

## Paso 5: Comprobar la configuración y reiniciar SSH

Primero verifica que no hay errores de sintaxis:

```bash
sudo sshd -t
```

- **Si no muestra nada**, todo está bien.
- **Si muestra un error**, corrígelo antes de continuar.

Ahora reinicia el servicio para aplicar los cambios:

```bash
sudo systemctl restart ssh
```


En algunas distribuciones (CentOS, Rocky, AlmaLinux, Fedora) el servicio se llama `sshd`:

```bash
sudo systemctl restart sshd
```

---

## Paso 6: Verificar que todo funciona

Sigue sin cerrar la sesión original. Desde **otra terminal nueva** haz estas dos pruebas:

**Prueba 1: con clave (debe funcionar)**

```bash
ssh usuario@192.168.0.1
```

Debes entrar sin problemas.

**Prueba 2: forzando contraseña (debe fallar)**

```bash
ssh -o PubkeyAuthentication=no usuario@192.168.0.1
```

Debe responder algo como:

```text
usuario@192.168.0.1: Permission denied (publickey).
```

Si ves ese mensaje, **¡enhorabuena!** Tu servidor ya solo acepta claves SSH.

---

## Resumen rápido

```bash
# 1. En tu ordenador: crear la clave
ssh-keygen -t ed25519 -C "mi-clave"

# 2. Copiar la clave pública al servidor
ssh-copy-id usuario@192.168.0.1

# 3. Probar el acceso con clave
ssh usuario@192.168.0.1

# 4. En el servidor: editar la configuración
sudo vim /etc/ssh/sshd_config
#    PasswordAuthentication no
#    KbdInteractiveAuthentication no
#    PubkeyAuthentication yes

# 5. Comprobar y reiniciar
sudo sshd -t
sudo systemctl restart ssh
```

---

## Problemas frecuentes

| Problema | Posible causa | Solución |
|----------|---------------|----------|
| Sigue pidiendo contraseña tras el Paso 3 | Permisos incorrectos en el servidor | `chmod 700 ~/.ssh && chmod 600 ~/.ssh/authorized_keys` |
| `Permission denied (publickey)` al probar con clave | La clave pública no está en `authorized_keys` o es de otro usuario | Repite el Paso 2 con el usuario correcto |
| Después de reiniciar sigue dejando entrar con contraseña | Un archivo de `sshd_config.d/` lo está reactivando | Revisa el Paso 4 (apartado de archivos que pisan la configuración) |
| Me he quedado sin acceso | Cerraste la sesión antes de verificar | Si dispones de una **consola web (VNC/rescue)** proporcionado por tu proveedor, desde aquí podrás recuperarlo. |
| `sshd -t` muestra un error | Error de escritura en `sshd_config` | Revisa la línea que indica el error |

---

## Buenas prácticas

- **Haz copia de seguridad de tu clave privada** (`id_ed25519`) en un lugar seguro. Si la pierdes y has desactivado la contraseña, perderás el acceso.
- **Usa passphrase** en tu clave privada.
- **No subas nunca tu clave privada** a GitHub, correo, chats ni nubes públicas.
- Usa **una clave distinta por ordenador**. Si pierdes uno, solo tienes que borrar su línea de `authorized_keys`.
