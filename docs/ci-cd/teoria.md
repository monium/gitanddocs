---
icon: lucide/refresh-cw
---

# Integración y despliegue continuos

Cuando varias personas modifican el mismo proyecto, probar y publicar a mano cada cambio es lento y propenso a errores. **CI/CD** automatiza ese camino: cada cambio que se sube a Git se comprueba y, si todo va bien, se prepara o se publica sin intervención manual.

Fuente de referencia: [CI/CD: integración y despliegue continuos](https://dpl-f1facc.gitlab.io/ut01/teoria/ci-cd/) (curso DPL, UT1).

!!! note "Requisito previo"
    Conviene dominar el flujo de [Git y GitHub](../trabajando-con-git.md) (ramas y *pull requests*), porque el CI se apoya en él.

## CI, entrega continua y despliegue continuo

Las siglas CI/CD agrupan tres ideas distintas:

| Práctica | Qué automatiza | ¿Quién decide publicar? |
|---|---|---|
| **Integración continua (CI)** | Compilar y probar cada cambio que se sube | Nadie: solo se verifica |
| **Entrega continua** | Todo lo anterior, dejando el resultado listo para publicar | Una persona, con una aprobación |
| **Despliegue continuo** | Todo lo anterior, incluida la publicación en producción | Nadie: es automático |

```text
push o PR → CI (compila y prueba) → artefacto listo → ¿aprobación humana? → producción
```

- **CI** busca detectar errores en minutos, no semanas después.
- **Entrega continua** significa que *siempre* se puede publicar, pero se elige cuándo.
- **Despliegue continuo** exige pruebas muy fiables, porque nadie revisa antes de llegar a producción.

## El pipeline

Un *pipeline* es la secuencia de fases que recorre un cambio hasta su despliegue. Si una fase falla, **las siguientes no se ejecutan** y el cambio queda marcado en rojo.

| Fase | Qué hace | Ejemplo |
|---|---|---|
| **Source** | Descarga el código del commit | `actions/checkout` |
| **Build** | Instala dependencias o compila | `pip install -r requirements.txt`, `npm ci` |
| **Test** | Ejecuta pruebas unitarias y de integración | `pytest`, `npm test` |
| **Package** | Genera el artefacto | imagen Docker, `.war`, ZIP |
| **Deploy** | Publica en un entorno | `docker push`, `kamal deploy` |

!!! tip "Fallo rápido"
    Pon primero lo más barato y rápido (análisis de estilo, compilación) y después lo costoso. Así un error simple se descubre en segundos y no gasta minutos de pruebas largas.

## Entornos

Un entorno es una copia del sistema donde se ejecuta el software en un momento del ciclo.

| Entorno | Para qué sirve | Quién lo usa |
|---|---|---|
| **Local** | Desarrollar y probar en tu máquina | La persona que programa |
| **Integración** | Ejecutar el *pipeline* y las pruebas automáticas | El propio CI |
| **Staging** (preproducción) | Validar el despliegue con datos parecidos a los reales | El equipo y QA |
| **Producción** | Lo que ven los usuarios finales | Los usuarios |

La regla de oro es que **los entornos se parezcan lo máximo posible**. Docker ayuda mucho: la misma imagen funciona igual en todos. Un ejemplo con dos entornos reales lo tienes en [Desplegar task-api con Kamal](../despliegue/task-api-kamal.md).

## Artefactos

Un **artefacto** es el resultado empaquetado y versionado del *pipeline*: una imagen Docker, un `.war`, un ZIP o un binario. Se guarda en un registro (Docker Hub, GitHub Container Registry, Nexus...) y se identifica por una versión o por el hash del commit.

Ventajas:

- **Promoción:** el mismo artefacto probado en *staging* es el que va a producción, sin recompilar.
- **Trazabilidad:** se sabe qué versión hay en cada entorno y de qué commit sale.
- **Reversibilidad:** volver atrás es desplegar el artefacto anterior.

## Ramas y CI

La estrategia de ramas determina cuándo se dispara cada cosa:

- **Ramas por funcionalidad (GitHub Flow):** cada *pull request* ejecuta las pruebas. Solo al fusionar en `main` se construye y publica el artefacto. Es lo que usamos en esta clase.
- **Trunk-based:** cada `push` a `main` dispara el *pipeline* y un despliegue frecuente. Lo que aún no está terminado se oculta con *feature flags*.

Publicar **solo desde `main`** evita generar artefactos de ramas que nunca llegarán a producción.

## Herramientas

| Herramienta | Modelo | Dónde se configura |
|---|---|---|
| **Jenkins** | Servidor propio, ampliable con *plugins* | `Jenkinsfile` |
| **GitHub Actions** | Servicio en la nube, integrado con los PR | `.github/workflows/*.yml` |
| **GitLab CI/CD** | Integrado en GitLab | `.gitlab-ci.yml` |

Todas comparten la idea de que un **archivo versionado en el repositorio** define *cuándo* y *qué* se ejecuta. Como vive en Git, el *pipeline* también se revisa con *pull requests*.

La siguiente página usa GitHub Actions.

## Secretos y buenas prácticas

- **Nunca escribas contraseñas, tokens o claves en el YAML.** Se versionarían y quedarían expuestos. Usa los secretos del repositorio (`secrets.NOMBRE`), que se inyectan al ejecutar.
- Los secretos no aparecen en los registros, pero **no los imprimas** con `echo`.
- Da **permisos mínimos** al *pipeline* con `permissions:`.
- **Fija las versiones** de las acciones (`@v4`) para que no cambien sin avisar.
- Trata el *pipeline* como código y revísalo con *pull requests*.

## Autoevaluación

??? question "1. ¿Qué diferencia hay entre integración continua, entrega continua y despliegue continuo?"
    La integración continua verifica automáticamente cada cambio (compila y prueba). La entrega continua además deja el artefacto listo para publicar, pero una persona aprueba el despliegue. El despliegue continuo publica en producción automáticamente, sin intervención.

??? question "2. ¿Cuáles son las fases típicas de un pipeline y por qué se ejecutan primero las más rápidas?"
    Source (descargar el código), build, test, package y deploy. Se ponen primero las baratas (lint, compilación) para fallar rápido: un error simple se detecta enseguida y no se malgasta tiempo ni recursos en pruebas largas.

??? question "3. ¿Qué es un entorno y cuáles son los habituales?"
    Es una copia del sistema donde se ejecuta el software en una fase del ciclo. Los habituales son local, integración, *staging* (preproducción) y producción. Deben parecerse lo máximo posible.

??? question "4. ¿Qué es un artefacto y por qué se promueve el mismo de un entorno a otro?"
    Es el resultado empaquetado y versionado del *pipeline* (por ejemplo, una imagen Docker). Se promueve el mismo para garantizar que lo que se probó es exactamente lo que se publica, con trazabilidad por commit y posibilidad de volver atrás.

??? question "5. ¿Por qué se publica solo desde `main` en un flujo con ramas por funcionalidad?"
    Porque las ramas de funcionalidad pueden no llegar nunca a producción. Si solo se publica desde `main`, no se generan artefactos de cambios que no se van a usar, y cada artefacto corresponde a código ya revisado.

??? question "6. Nombra tres herramientas de CI/CD y dónde se guarda su configuración."
    Jenkins (`Jenkinsfile`), GitHub Actions (`.github/workflows/*.yml`) y GitLab CI/CD (`.gitlab-ci.yml`).

??? question "7. ¿Qué diferencia hay entre `on`, `jobs` y `steps` en GitHub Actions?"
    `on` define los eventos que disparan el *workflow*. `jobs` son los trabajos, cada uno en su propia máquina virtual. `steps` son los pasos secuenciales dentro de cada trabajo.

??? question "8. ¿Por qué no se deben escribir secretos en el YAML y cómo se usan?"
    Porque el archivo se versiona y los secretos quedarían expuestos en el historial. En GitHub Actions se guardan en los secretos del repositorio y se referencian como `secrets.NOMBRE`, de modo que se inyectan solo al ejecutar.

## Qué deberías saber hacer

- Distinguir CI, entrega continua y despliegue continuo.
- Describir las fases de un *pipeline* y asociarlas a comandos reales.
- Explicar los entornos local, integración, *staging* y producción.
- Saber qué es un artefacto y por qué se versiona.
- Usar secretos y permisos mínimos.
