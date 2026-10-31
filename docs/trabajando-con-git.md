---
icon: lucide/git-branch
---

# Git y Github

Git es un sistema de control de versiones que permite guardar y controlar los cambios realizados en un proyecto.

Github es una plataforma que permite alamcenar respositorios Git en Internet y trabajaar con otras persoans.

## 1. Fork del respositorio

Un fork es una copia de un repositorio que se crea en nuestra propia cuenta de GitHub.

Por ejemplo, podemos hacer un fork del repositorio de otra persona para poder trabajar sobre nuestra propia copia sin modificar directamente el repositorio original.

En Github

En este ejercicio, el repositorio original pertenece al profesor:
```
https://github.com/profe_usuario/gitanddocs
```

Después de hacer el fork, tenemos nuestra propia copia:
```
https://github.com/tu_usuario/gitanddocs
```

## 2. Clonar el respositorio

Una vez creado el fork, debemos descargar el repositorio en nuestro ordenador.

En Bash


Para ello utilizamos el comando:

```bash
git clone https://github.com/tu_usuario/gitanddocs.git
```

## 3. Configurar upstream

Cuando clonamos nuestro repositorio, Git configura automáticamente un remoto llamado `origin`.

`origin` hace referencia a nuestro repositorio de GitHub:

https://github.com/tu_usuario/gitanddocs.git

Para poder obtener los cambios del repositorio original del profesor, añadimos otro remoto llamado `upstream`:

```bash
git remote add upstream https://github.com/monium/gitanddocs.git
```

De esta forma tenemos dos remotos:

- `origin:` Nuestro respositorio
- `upstream:`  Repositorio original del profesor

 Hacemos que Git utilice estos nombres para identificar repositorios remotos en local de forma cómoda para nosotros.

 ## 4. Obtener cambios del repositorio original y actualizar nuestra rama `main`

Para obtener información sobre los cambios que existen en el repositorio del profesor utilizamos:

```bash
git fetch upstream
```
Recordatorio: `fetch` obtienes los cambios del repositorio remoto y los guarda en nuestro repositorio local, pero no modifica nuestra rama.

Después de obtener los cambios con `fetch`, podemos incorporarlos a nuestra rama `main` utilizando:

```bash
git merge upstream/main
```
Importante!: Tenemos que estar situados en nuestra rama `main` antes de ejecutar el `merge`. Simplemente utilizando `git branch` aparecerá el nombre de nuestra rama.

## 5. Crear y subir a Github una rama de trabajo

Una vez tenemos nuestra rama `main` actualizada, podemos crear una rama independiente para realizar nuestros cambios.

Para crearla y cambiar a ella utilizamos:

```bash
git switch -c nombre-rama
```

El comando `switch` es cambiar y `-c` indica que queremos crear una rama nueva.


Después de crear nuestra rama de trabajo, podemos subirla a nuestro repositorio de GitHub utilizando:

```bash
git push -u origin nombre-rama
```

En este comando:
- `push:` sube nuestros cambios al repositorio remoto.
- `-u:` establece la relación entre nuestra rama local y la rama remota.
- `origin:` indica que queremos subirlos a nuestro repositorio de GitHub.
- `tu-rama:` indica la rama que queremos subir.

## 6. Guardar cambios con un commit

Cuando realizamos cambios en los archivos del proyecto, Git detecta que esos archivos han sido modificados.

+ Podemos comprobar el estado del proyecto con:

```bash
git status
```

+ Para preparar los archivos que queremos incluir en el commit utilizamos:

```bash
git add .
```
+ Después creamos el commit:

```bash
git commit -m "Descripción de los cambios"
```
Recuerda: `git commit`todavía no sube nada a Github. Todo se queda en nuestro repositorio lcoal.

+ Por último subirlo todo a Github:

```bash
git push
```

## 7. Pull Request

Una vez hemos realizado nuestros cambios, creado los commits y subido nuestra rama a GitHub, podemos crear un Pull Request.

Un Pull Request (PR) es una solicitud para que los cambios realizados en una rama sean revisados y, si se aceptan, incorporados a otra rama.

En nuestro caso, queremos enviar nuestros cambios desde nuestra rama `tu-rama` hacia la rama `main` del repositorio original del profesor.

El proceso sería:

```text
Nuestro ordenador
      │
      │ git push
      ↓
Nuestro GitHub
      │
      │ Pull Request
      ↓
GitHub del profesor
      │
      ↓
     main
```
Para crear el Pull Request:
1. Subimos nuestra rama a nuestro repositorio con git push.
2. Entramos en nuestro repositorio de GitHub.
3. Seleccionamos la opción para crear un Pull Request.
4. Comprobamos que la rama de origen es nuestra rama jorgeM.
5. Comprobamos que el repositorio y la rama de destino son el repositorio original del profesor y su rama main.
6. Añadimos una descripción de los cambios realizados.
7. Creamos el Pull Request.


## Siguientes pasos

Cuando domines este flujo, continúa con la sección **Git intermedio**:

- [Merge, rebase y conflictos](git/merge-rebase-conflictos.md)
- [Deshacer cambios](git/deshacer-cambios.md)
- [.gitignore y tags](git/gitignore-tags.md)
