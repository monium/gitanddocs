---
icon: lucide/tag
---

# .gitignore, tags y releases

## `.gitignore`

El archivo `.gitignore` indica a Git qué archivos **no debe versionar**: dependencias, entornos virtuales, archivos temporales, claves, etc.

Se crea en la raíz del repositorio. Ejemplo para este proyecto:

```text
# Entorno virtual de Python
.venv/

# Web generada por Zensical
site/

# Archivos del sistema
.DS_Store

# Secretos
.env
```

### Patrones habituales

| Patrón | Qué ignora |
|---|---|
| `*.log` | Todos los archivos `.log` |
| `carpeta/` | Esa carpeta y su contenido |
| `!importante.log` | Excepción: sí se versiona |
| `/archivo` | Solo en la raíz |

### Ya estaba subido y quiero ignorarlo

`.gitignore` solo afecta a archivos **nuevos**. Para dejar de seguir uno que ya está en el repositorio:

```bash
git rm --cached archivo
git commit -m "Deja de versionar archivo"
```

`--cached` lo quita de Git pero lo conserva en tu disco.

!!! danger "Secretos"
    Si subiste una contraseña o clave, ignorarla después **no basta**: queda en el historial. Hay que cambiar esa clave.

## Tags

Un tag es una **etiqueta fija** sobre un commit, normalmente para marcar versiones (`v1.0`, `v2.1.3`).

```bash
git tag v1.0                          # ligero
git tag -a v1.0 -m "Primera versión"  # anotado (recomendado)
git tag                               # listar
git show v1.0                         # ver detalles
```

Los tags **no se suben solos**:

```bash
git push origin v1.0
git push origin --tags
```

Borrar un tag:

```bash
git tag -d v1.0
git push origin --delete v1.0
```

### Versionado semántico

Formato `MAYOR.MENOR.PARCHE`:

- `PARCHE`: arreglos pequeños (`1.0.1`).
- `MENOR`: novedades compatibles (`1.1.0`).
- `MAYOR`: cambios que rompen compatibilidad (`2.0.0`).

## Releases en GitHub

Una *release* es un tag con descripción, notas y archivos adjuntos.

1. Entra en el repositorio y pulsa **Releases**.
2. Pulsa **Draft a new release**.
3. Elige o crea un tag (por ejemplo `v1.0`).
4. Escribe el título y las notas (puedes usar *Generate release notes*).
5. Pulsa **Publish release**.
