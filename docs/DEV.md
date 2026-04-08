# Desarrollo local – PSIRA

Este documento explica **cómo trabajar en el proyecto PSIRA en un entorno local**, sin afectar los servidores de staging o producción y **sin publicar imágenes**.

El objetivo es que cualquier integrante del equipo pueda:

* levantar el sistema localmente,
* hacer cambios en frontend o backend,
* probarlos,
* y abrir un Pull Request hacia `develop`.

---
## Índice

- [1. Concepto general (importante leer)](#1-concepto-general-importante-leer)
- [2. Requisitos](#2-requisitos)
  - [Sistema operativo](#sistema-operativo)
  - [Software necesario](#software-necesario)
- [3. Estructura de carpetas esperada](#3-estructura-de-carpetas-esperada)
- [4. Variables de entorno (solo una vez)](#4-variables-de-entorno-solo-una-vez)
- [5. Levantar el entorno de desarrollo](#5-levantar-el-entorno-de-desarrollo)
- [6. Accesos locales](#6-accesos-locales)
- [7. Empezamos con Git (antes de empezar a editar)](#7-empezamos-con-git-antes-de-empezar-a-editar)
  - [Ramas](#ramas)
  - [Crear una rama de trabajo propia cada vez que vas a editar por primera vez un repositorio](#crear-una-rama-de-trabajo-propia-cada-vez-que-vas-a-editar-por-primera-vez-un-repositorio)
  - [Crear y cambiar de rama de trabajo](#crear-y-cambiar-de-rama-de-trabajo)
  - [Subir la rama de trabajo a GitHub](#subir-la-rama-de-trabajo-a-github)
- [8. Ver cambios en el sistema](#8-ver-cambios-en-el-sistema)
- [9. Logs y debugging](#9-logs-y-debugging)
- [10. Pull Requests](#10-pull-requests)
  - [10.0. Verificar y guardar cambios antes de continuar](#100-verificar-y-guardar-cambios-antes-de-continuar)
    - [Ver estado de los archivos](#ver-estado-de-los-archivos)
    - [Agregar cambios al próximo commit](#agregar-cambios-al-próximo-commit)
    - [Guardar cambios en un commit](#guardar-cambios-en-un-commit)
    - [Estado ideal antes de seguir](#estado-ideal-antes-de-seguir)
  - [10.1. Antes de abrir el Pull Request, actualizar la rama con `develop`](#101-antes-de-abrir-el-pull-request-actualizar-la-rama-con-develop)
    - [Opción recomendada: `merge`](#opción-recomendada-merge)
    - [Si aparecen conflictos](#si-aparecen-conflictos)
  - [10.2. Abrir el Pull Request en GitHub](#102-abrir-el-pull-request-en-github)
  - [10.3. Revisión y merge](#103-revisión-y-merge)
  - [10.4. Después de que el Pull Request fue aceptado](#104-después-de-que-el-pull-request-fue-aceptado)
    - [Eliminar la rama de trabajo](#eliminar-la-rama-de-trabajo)
- [Aclaraciones importantes](#aclaraciones-importantes)
  - [`checkout`](#checkout)
  - [`add`, `commit` y `push`](#add-commit-y-push)
  - [Trabajar en carpetas distintas no evita actualizar la rama](#trabajar-en-carpetas-distintas-no-evita-actualizar-la-rama)
  - [Si dos Pull Requests se abren casi al mismo tiempo](#si-dos-pull-requests-se-abren-casi-al-mismo-tiempo)
- [11. Errores comunes](#11-errores-comunes)
- [12. Qué **NO** hacer](#12-qué-no-hacer)
- [13. Filosofía del entorno](#13-filosofía-del-entorno)
---

## 1. Concepto general (importante leer)

El proyecto se organiza en **tres repositorios separados**:

* `psira-docker` → infraestructura (Docker, base de datos, proxy, etc.)
* `psira-frontend` → aplicación frontend
* `psira-backend` → aplicación backend
* `psira-shiny` → aplicación shiny

En **producción y staging**, el sistema corre usando **imágenes publicadas**.
En **desarrollo local**, el sistema corre usando **el código fuente local**, sin subir nada a ningún registry.

Esto se logra usando **dos archivos de Docker Compose**:

* `docker-compose.yml` → base (deploy)
* `docker-compose.dev.yml` → override para desarrollo local

---

## 2. Requisitos

### Sistema operativo

* Windows 11 (recomendado)
* También funciona en Linux/macOS con mínimos ajustes

### Software necesario

Instalar **una sola vez**:

1. **Docker Desktop** (Debe estar activo para usarlo desde Terminal)

   * Activar *Use WSL 2 based engine (desde la configuracion asegurarse que todo este marcado)*
2. **WSL2 con Ubuntu**
```bash
wsl --install
# Acceder a la consola Ubuntu
ubuntu
```

3. **Git**
4. **VS Code** (recomendado)

   * Extensión *WSL* (opcional pero muy útil)

Verificar instalación:

```bash
docker version
docker compose version
docker ps
# si da permission denied:
sudo usermod -aG docker $USER
newgrp docker
```

---

## 3. Estructura de carpetas esperada

En tu computadora (preferentemente dentro de WSL), crear una carpeta común, por ejemplo:

```text
~/eipsi/psira/
  psira-docker/
  psira-frontend/
  psira-backend/
  psira-shiny/
```

Clonar los repositorios en esa estructura:

```bash
# Para Windows
ubuntu
mkdir -p ~/eipsi/psira
cd ~/eipsi/psira
git clone -b develop https://github.com/EIPSI/psira-docker.git
git clone -b develop https://github.com/EIPSI/psira-frontend.git
git clone -b develop https://github.com/EIPSI/psira-backend.git
sudo git clone https://github.com/EIPSI/psira-shiny.git
```

Durante el proceso, Git va a pedir credenciales:

```text
Username for 'https://github.com': [USUARIO_GITHUB]
Password for 'https://[USUARIO_GITHUB]@github.com': [TOKEN_GITHUB]
```

**Importante:** en el campo `Password` no tenés que ingresar tu clave de GitHub, sino un **Personal Access Token (PAT)** de GitHub con permisos para acceder al repositorio privado.

---
> ⚠️ **Importante:**
> Los nombres y la ubicación relativa importan porque `docker-compose.dev.yml` asume esta estructura.
---


Abrir carpetas desde la terminal
```bash
# Para Windows
ubuntu
# Te pones dentro de la carpeta que queres abrir:
explorer.exe .
# Luego copias el path para abrir la carpeta con VS Code o cualquier editor
```

---

## 4. Variables de entorno (solo una vez)

Dentro del repo **`psira-docker`**:

```bash
cd psira-docker
cp .env.local.example .env
```

El archivo `.env` está ignorado por `.gitignore` y es solo para configuración local. No lo compartas ni lo fuerces a commitear (`git add -f`). Para compartir configuración usamos `.env.local.example`.

---

## 5. Levantar el entorno de desarrollo

Desde la carpeta `psira-docker`:

```bash
docker compose \
  -f docker-compose.yml \
  -f docker-compose.dev.yml \
  up -d --build
```

Qué hace este comando:

* levanta base de datos, redis, mongo, caddy, etc.
* build-ea frontend y backend **desde el código local**
* corre todo en contenedores
* no publica imágenes

La primera vez puede demorar varios minutos.

Ver el estado de los servicios:
```bash
docker ps
# Si alguno no esta UP podemos chequear problemas en cada uno con
docker logs --tail 200 <nmbre-del-servicio-como-esta-en-col-NAME>
```

---

## 6. Accesos locales

Una vez levantado el entorno:

* **Frontend (HTTP):**
  [http://psira.localhost:8080](http://psira.localhost:8080)

* **Frontend (HTTPS):**
  [https://psira.localhost:8443](https://psira.localhost:8443) (si el navegador redirige a HTTPS o si querés probar con TLS local)

* **GraphQL (HTTP):**
  [http://psira.localhost:8080/graphql](http://psira.localhost:8080/graphql)

* **GraphQL (HTTPS):**
  [https://psira.localhost:8443/graphql](https://psira.localhost:8443/graphql)

* **Shiny:**
  [http://psira.localhost:3838/](http://psira.localhost:3838/)

* **Base de datos:**
  No es necesario acceder directamente para desarrollo habitual.

---

## 7. Empezamos con Git (antes de empezar a editar)

### Ramas

* `main` → producción (no tocar)
* `develop` → integración/staging
* `nombre/tarea` → ramas de trabajo individual

### Crear una rama de trabajo propia cada vez que vas a editar por primera vez un repositorio

```bash
# Entras al repo local y decis que estas en esta rama
git checkout develop
# Actualizas la ultima version de la rama
git pull origin develop
```

### Crear y cambiar de rama de trabajo

```bash
# Creas tu propia rama y cambiar a ella
git checkout -b nombre/descripcion-corta
# Solo cambiar de rama de trabajo
git checkout nombre-rama
```

Ejemplo:

```bash
git checkout -b juan/fix-questionnaire-error
```

### Subir la rama de trabajo a GitHub

```bash
# Agregar todos los archivos del repo local a la nueva version
git add .
# Hacer commit para describir cambios
git commit -m "creacion de rama para realizar ... "
# Subir la rama de trabajo a GitHub
git push -u origin nombre/descripcion-corta
```

Esto hace dos cosas:

* sube la rama local a GitHub
* deja configurada la rama remota asociada, para que después alcance con `git push` y `git pull`

⚠️ Este comando **no mergea** la rama con `develop`.
Solo publica la rama en GitHub para que pueda abrirse un Pull Request.


> ### Una vez creada, subida y en la rama propia: se pueden realizar todos los cambios que uno quiera.

[Continuar con el flujo de trabajo con Git en la sección **10. Pull Requests**](#10-pull-requests)

---

## 8. Ver cambios en el sistema

Este entorno **NO usa hot reload** por ahora.

Flujo normal:

1. editar archivos en frontend o backend
2. volver a la carpeta `psira-docker`
3. ejecutar:

```bash
docker compose \
  -f docker-compose.yml \
  -f docker-compose.dev.yml \
  up -d
```
4. o reiniciá solo el servicio:

```bash
docker compose \
  -f docker-compose.yml \
  -f docker-compose.dev.yml \
  restart psira-backend
```

5. refrescar el navegador (o **borrar caché** / datos del sitio)

* **Mozilla Firefox**
  1. Abrir `about:preferences#privacy`.
  2. **Cookies y datos del sitio** → **Administrar datos…**.
  3. Buscar `psira.localhost` y `localhost`.
  4. **Eliminar seleccionados** → **Guardar cambios**.
  5. Recargar la página (mejor en ventana privada para test).
* **Google Chrome**
  1. Abrir `chrome://settings/siteData`.
  2. Buscar `psira.localhost` y `localhost`.
  3. **Eliminar** todas las entradas.
  4. Abrir DevTools (**F12**) → **Application** → **Storage** → **Clear site data**.
  5. Probar nuevamente.
* **Opera**
  1. Abrir `opera://settings/siteData`.
  2. Buscar `psira.localhost` y `localhost`.
  3. **Remove all shown**.
  4. DevTools (**F12**) → **Application** → **Storage** → **Clear site data**.
  5. Probar en ventana privada.

Esto garantiza que todos trabajen en un entorno estable e idéntico.

---

## 9. Logs y debugging

Ver logs de un servicio específico:

```bash
docker compose logs -f psira-backend
docker compose logs -f psira-frontend
docker compose logs -f caddy
```

Detener el entorno:

```bash
docker compose down
```

---

Sí. Además, en ese bloque hay una confusión importante: **`git push -u origin nombre/descripcion-corta` no mergea con `develop`**.
Eso solo **sube la rama de trabajo a GitHub** y la deja vinculada al remoto. El merge ocurre después, vía Pull Request.

Te propongo una versión más correcta y más clara del instructivo:

---

## 10. Pull Requests

Cuando el cambio esté listo, hay que **actualizar la rama de trabajo al remoto (GitHub)** y luego abrir un Pull Request hacia `develop`.


### 10.0. Verificar y guardar cambios antes de continuar

Antes de actualizar la rama o subir cambios, es importante verificar el estado del repositorio y guardar los cambios realizados.

#### Ver estado de los archivos

```bash
git status
```

Este comando muestra:

* archivos modificados
* archivos nuevos (untracked)
* archivos listos para commit (staged)

---

#### Agregar cambios al próximo commit

Para incluir archivos en el commit:

```bash
git add nombre-del-archivo
```

O para agregar todos los cambios:

```bash
git add .
```

`git add` indica qué cambios van a formar parte del próximo commit.

---

#### Guardar cambios en un commit

```bash
git commit -m "descripción breve del cambio"
```

Esto guarda los cambios en la historia local del repositorio.

---

⚠️ **Importante**:

* Si no se hace `git add`, los cambios no entran en el commit.
* Si no se hace `git commit`, los cambios no pueden subirse con `git push`.
* `git push` solo sube commits, no archivos sin guardar.

---

#### Estado ideal antes de seguir

Antes de hacer `merge` o `push`, el repositorio debería estar limpio:

```bash
git status
```

Debe mostrar:

```txt
nothing to commit, working tree clean
```

Esto indica que todos los cambios están correctamente guardados.


### 10.1. Antes de abrir el Pull Request, actualizar la rama con `develop`

Si durante el trabajo otra persona hizo cambios y `develop` avanzó, la rama propia puede quedar desactualizada.
Para evitar conflictos o cambios inesperados en el Pull Request, primero hay que traer la última versión de `develop` a la rama de trabajo.

#### Opción recomendada: `merge`

```bash
git checkout develop
git pull origin develop
git checkout nombre/descripcion-corta
git merge develop
```

Esto significa:

1. cambiar a la rama `develop`
2. actualizar la copia local de `develop` con lo que hay en GitHub
3. volver a la rama de trabajo
4. incorporar los cambios nuevos de `develop` a esa rama

Si no hay conflictos, después subir nuevamente la rama:

```bash
git push
```

#### Si aparecen conflictos

Git lo va a indicar en la terminal. Después revisar:

```bash
git status
```

Los archivos en conflicto deben resolverse manualmente.
Una vez resueltos:

```bash
git add nombre-del-archivo
git commit
git push
```

⚠️ Los conflictos aparecen cuando Git no puede decidir automáticamente qué versión conservar. Esto suele ocurrir si dos ramas modificaron la misma parte de un archivo, o si una rama borró algo que otra modificó.

---

### 10.2. Abrir el Pull Request en GitHub

Una vez subida y actualizada la rama:

1. ir a GitHub
2. abrir un Pull Request
3. elegir como rama base: `develop`
4. elegir como rama comparada: `nombre/descripcion-corta`

En la descripción del Pull Request indicar:

* qué se cambió
* cómo probarlo
* si hay algo que pueda romper funcionamiento previo
* cualquier detalle que la persona revisora deba saber

---

### 10.3. Revisión y merge

⚠️ Solo la **persona responsable del proyecto** puede hacer merge a `develop` o `main`.

Esto permite:

* revisar los cambios antes de integrarlos
* detectar problemas
* mantener `develop` y `main` ordenadas
* evitar que se suban cambios incompletos o no revisados

---

### 10.4. Después de que el Pull Request fue aceptado

Una vez que la rama fue mergeada en `develop`, la rama de trabajo ya no debe seguir usándose para nuevas tareas.

#### Eliminar la rama de trabajo

La **persona responsable del proyecto** se encarga de borrar la rama en GitHub, pero ***usted debe eliminarla a nivel local***.

Borrado local:

```bash
git branch -d nombre/descripcion-corta
```

Luego, para empezar una tarea nueva:

```bash
git checkout develop
git pull origin develop
git checkout -b nueva-rama
```

⚠️ No reutilizar ramas viejas para tareas nuevas.
Cada tarea debe tener su propia rama.

---

## Aclaraciones importantes

### `checkout`

`git checkout` sirve para cambiar de rama.

Por ejemplo:

```bash
git checkout develop
```

significa: cambiar a la rama `develop`.

Y:

```bash
git checkout -b nombre/descripcion-corta
```

significa:

* crear una rama nueva
* cambiarse a esa rama inmediatamente

---

### `add`, `commit` y `push`

Antes de poder subir cambios, primero hay que guardarlos en un commit.

Flujo normal:

```bash
git add .
git commit -m "mensaje descriptivo"
git push
```

* `git add` selecciona qué cambios entran al próximo commit
* `git commit` guarda esos cambios en la historia local
* `git push` sube esos commits a GitHub

⚠️ `git push` no sube archivos sueltos: sube commits.

---

### Trabajar en carpetas distintas no evita actualizar la rama

Aunque cada persona trabaje en una carpeta distinta, igual debe actualizar su rama con `develop` antes del Pull Request.

Git no hace pull de una carpeta aislada, sino de commits y ramas completas.

---

### Si dos Pull Requests se abren casi al mismo tiempo

Aunque dos personas trabajen sobre archivos distintos, si uno de los Pull Requests entra primero, el otro puede quedar desactualizado respecto de `develop`.

Por eso, antes de mergear un Pull Request, la rama debe actualizarse otra vez con `develop` si hubo cambios recientes.

---

## 11. Errores comunes

### ❌ No abre el sitio

* Verificar que Docker esté corriendo
* Ver `docker compose logs caddy`

### ❌ Error de build

* Ver logs del servicio correspondiente
* Asegurarse de que frontend/backend tengan su `Dockerfile`

### ❌ Cambios no se reflejan

* Asegurarse de haber corrido `--build`
* Refrescar el navegador

---

## 12. Qué **NO** hacer

* ❌ No subir imágenes a GHCR
* ❌ No trabajar directamente en `main` o `develop`
* ❌ No subir `.env`, `.data`, `.backups`
* ❌ No modificar `docker-compose.yml` base sin avisar

---

## 13. Filosofía del entorno

Este setup prioriza:

* reproducibilidad
* bajo costo cognitivo
* evitar “en mi máquina funciona”
* proteger producción

La optimización (hot reload, dev servers) vendrá después.
