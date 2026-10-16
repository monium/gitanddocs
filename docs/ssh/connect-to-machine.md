---
icon: lucide/terminal
---

# Conexión SSH mediante clave pública y deshabilitación de contraseña

Esta guía describe cómo conectarse a una máquina remota por SSH usando autenticación con clave pública y, después, deshabilitar el inicio de sesión por contraseña.

## Requisitos

- Acceso inicial al servidor remoto (usuario y contraseña o clave existente).
- Permisos de `sudo` en el servidor remoto.
- Cliente SSH instalado en el equipo local.

## 1. Generar un par de claves SSH en el equipo local

Si aún no tienes una clave SSH,  créala. Para ello, consulta el documento aparte: [create-ssh-key.md](create-ssh-key.md).

## 2. Copiar la clave pública al servidor remoto

Usa `ssh-copy-id`:

```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub usuario@host_remoto
```

Si `ssh-copy-id` no está disponible, copia la clave manualmente:

```bash
cat ~/.ssh/id_ed25519.pub | ssh usuario@host_remoto \
  "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

## 3. Probar la autenticación con clave

Verifica que puedes iniciar sesión sin contraseña:

```bash
ssh -i ~/.ssh/id_ed25519 usuario@host_remoto
```

Si funciona, ya tienes autenticación por clave.

## 4. Deshabilitar el inicio de sesión por contraseña en el servidor remoto

**Importante:** mantén abierta la sesión SSH actual mientras haces estos cambios. Así podrás revertirlos si algo falla.

Edita la configuración de SSH en el servidor remoto:

```bash
sudo nano /etc/ssh/sshd_config
```

Asegura o añade estas directivas:

```sshconfig
PubkeyAuthentication yes
PasswordAuthentication no
KbdInteractiveAuthentication no
ChallengeResponseAuthentication no
PermitEmptyPasswords no
```

También puedes crear un archivo de configuración adicional si tu distribución lo soporta:

```bash
sudo tee /etc/ssh/sshd_config.d/99-no-password.conf > /dev/null <<'EOF'
PubkeyAuthentication yes
PasswordAuthentication no
KbdInteractiveAuthentication no
ChallengeResponseAuthentication no
PermitEmptyPasswords no
EOF
```

## 5. Validar y aplicar la configuración

Antes de recargar, comprueba que no haya errores de sintaxis:

```bash
sudo sshd -t
```

Recarga el servicio SSH:

```bash
# Debian/Ubuntu
sudo systemctl reload ssh

# RHEL/CentOS/Fedora
sudo systemctl reload sshd
```

Verifica la configuración efectiva:

```bash
sudo sshd -T | grep -E 'passwordauthentication|pubkeyauthentication|kbdinteractiveauthentication|challengeresponseauthentication|permitemptypasswords'
```

## 6. Probar la nueva conexión

**No cierres la sesión actual.** Abre una nueva terminal y prueba:

```bash
ssh -i ~/.ssh/id_ed25519 usuario@host_remoto
```

Si la conexión por clave funciona y la contraseña ya no es aceptada, el cambio fue exitoso.

## Solución de problemas

- Revisa los logs del servidor:
  ```bash
  sudo journalctl -u ssh -e
  # o
  sudo journalctl -u sshd -e
  ```
- Verifica permisos en el servidor remoto:
  ```bash
  chmod 700 ~/.ssh
  chmod 600 ~/.ssh/authorized_keys
  ```
- Comprueba que la clave pública correcta esté en `~/.ssh/authorized_keys`.
- Si `sshd -t` muestra errores, corrige el archivo antes de recargar.
