---
icon: lucide/undo-2
---

# Deshacer cambios

Equivocarse en Git casi siempre tiene solución. Lo importante es saber **en qué fase** está el cambio.

```text
Directorio de trabajo → (git add) → Área de preparación → (git commit) → Repositorio local → (git push) → GitHub
```

## Descartar cambios sin commitear

Deshacer los cambios de un archivo (vuelve a la última versión commiteada):

```bash
git restore docs/index.md
```

!!! danger
    Esto **borra tus cambios sin posibilidad de recuperarlos**.

Sacar un archivo del área de preparación (deshace `git add`, conserva los cambios):

```bash
git restore --staged docs/index.md
```

## Corregir el último commit

Cambiar el mensaje o añadir un archivo olvidado:

```bash
git add archivo-olvidado.md
git commit --amend -m "Mensaje corregido"
```

!!! warning
    `--amend` reescribe el commit. Úsalo solo si todavía no has hecho `push`.

## Deshacer commits locales: `reset`

`reset` mueve la rama hacia atrás. Tiene tres modos:

| Comando | Commit | Área de preparación | Archivos |
|---|---|---|---|
| `git reset --soft HEAD~1` | se deshace | se conserva | se conservan |
| `git reset HEAD~1` (`--mixed`) | se deshace | se vacía | se conservan |
| `git reset --hard HEAD~1` | se deshace | se vacía | **se pierden** |

`HEAD~1` significa "un commit antes del actual".

```bash
git reset --soft HEAD~1    # quiero rehacer el commit
```

!!! danger
    No hagas `reset` de commits que ya subiste a GitHub y que otros pueden haber descargado.

## Deshacer commits ya publicados: `revert`

`revert` no borra historia: crea un **commit nuevo** que invierte los cambios de otro.

```bash
git log --oneline
git revert a1b2c3d
git push
```

Es la forma segura de deshacer algo ya compartido.

## Guardar cambios a medias: `stash`

Si necesitas cambiar de rama pero no quieres commitear lo que llevas:

```bash
git stash              # guarda y limpia el directorio de trabajo
git switch otra-rama
# ... trabajas ...
git switch mi-rama
git stash pop          # recupera lo guardado
```

Otros comandos útiles:

```bash
git stash list         # ver lo guardado
git stash drop         # descartar el último stash
```

## Recuperar lo "perdido": `reflog`

`git reflog` guarda un registro de dónde ha estado `HEAD`, incluso tras un `reset --hard`.

```bash
git reflog
git reset --hard HEAD@{2}
```

!!! tip
    Mientras no ejecutes limpiezas, un commit casi nunca se pierde del todo: casi siempre sale en el `reflog`.

## Resumen: qué usar

| Quiero... | Comando |
|---|---|
| Descartar cambios de un archivo | `git restore archivo` |
| Quitar un archivo del `add` | `git restore --staged archivo` |
| Arreglar el último commit (sin push) | `git commit --amend` |
| Deshacer commits locales | `git reset` |
| Deshacer un commit ya subido | `git revert` |
| Apartar trabajo temporalmente | `git stash` |
| Recuperar algo perdido | `git reflog` |
