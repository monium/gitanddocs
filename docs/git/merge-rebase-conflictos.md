---
icon: lucide/git-merge
---

# Merge, rebase y conflictos

Cuando trabajamos en una rama, antes o después hay que juntar su historia con la de otra (normalmente `main`). Git ofrece dos formas: `merge` y `rebase`.

## Merge

`merge` une dos ramas creando, si hace falta, un **commit de fusión** que conserva el historial tal como ocurrió.

```bash
git switch main
git merge mi-rama
```

```text
A---B---C   main
     \   \
      D---E---M   (M = commit de merge)
```

- Ventaja: no reescribe historia, es seguro.
- Inconveniente: el historial puede llenarse de commits de merge.

Si `main` no ha avanzado desde que creaste la rama, Git hace un *fast-forward*: solo mueve el puntero y no crea commit de merge.

## Rebase

`rebase` **recoloca** los commits de tu rama encima de la punta de otra, como si hubieras empezado a trabajar desde ahí.

```bash
git switch mi-rama
git rebase main
```

```text
Antes:                    Después:
A---B---C   main          A---B---C   main
     \                              \
      D---E   mi-rama                D'---E'   mi-rama
```

- Ventaja: historial lineal y limpio.
- Inconveniente: **reescribe commits** (D y E pasan a ser D' y E', con otros hashes).

!!! warning "Regla de oro del rebase"
    No hagas `rebase` de ramas que otras personas ya han descargado. Reescribir historia compartida causa problemas a todo el mundo.

## ¿Cuál usar?

| Situación | Recomendado |
|---|---|
| Actualizar tu rama de trabajo con los cambios de `main` antes de abrir el PR | `rebase` (o `merge`) |
| Integrar una rama ya publicada y compartida | `merge` |
| Quieres conservar exactamente lo que pasó | `merge` |
| Quieres un historial lineal | `rebase` |

En esta clase, como la guía de [Git y GitHub](../trabajando-con-git.md) ya usa `merge upstream/main`, puedes seguir con `merge` si dudas: es lo más seguro.

## Conflictos

Un conflicto aparece cuando dos ramas modifican **las mismas líneas** de un archivo y Git no sabe cuál conservar. No es un error: Git simplemente te pide que decidas.

```bash
git merge upstream/main
```

```text
Auto-merging docs/index.md
CONFLICT (content): Merge conflict in docs/index.md
Automatic merge failed; fix conflicts and then commit the result.
```

### Paso 1: ver qué archivos tienen conflicto

```bash
git status
```

Los archivos aparecen como `both modified` bajo *Unmerged paths*.

### Paso 2: abrir el archivo y leer los marcadores

```text
<<<<<<< HEAD
Texto de tu rama
=======
Texto de la otra rama
>>>>>>> upstream/main
```

- Entre `<<<<<<<` y `=======` está **tu versión**.
- Entre `=======` y `>>>>>>>` está **la versión que llega**.

### Paso 3: decidir y limpiar

Deja el texto final que quieras (una de las dos, o una mezcla) y **borra los tres marcadores**:

```text
Texto final acordado
```

!!! tip
    VS Code muestra botones sobre cada conflicto: *Accept Current*, *Accept Incoming* y *Accept Both*.

### Paso 4: marcar como resuelto y terminar

Con `merge`:

```bash
git add docs/index.md
git commit
```

Con `rebase`:

```bash
git add docs/index.md
git rebase --continue
```

### Cancelar si te pierdes

```bash
git merge --abort
git rebase --abort
```

Vuelves al estado anterior a empezar.

## Subir una rama rebaseada

Tras un `rebase`, tu rama local ya no coincide con la remota y `git push` será rechazado. Usa:

```bash
git push --force-with-lease
```

`--force-with-lease` es más seguro que `--force`: falla si alguien ha subido algo que tú no tienes.

## Cómo evitar conflictos

- Actualiza tu rama con `main` a menudo.
- Haz PRs pequeños que toquen pocos archivos.
- Avisa en clase si vas a editar un archivo compartido (`zensical.toml`, `index.md`).
