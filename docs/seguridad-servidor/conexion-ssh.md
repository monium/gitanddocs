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

