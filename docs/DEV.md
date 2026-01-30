# Desarrollo local – PSIRA

Este documento explica **cómo trabajar en el proyecto PSIRA en un entorno local**, sin afectar los servidores de staging o producción y **sin publicar imágenes**.

El objetivo es que cualquier integrante del equipo pueda:

* levantar el sistema localmente,
* hacer cambios en frontend o backend,
* probarlos,
* y abrir un Pull Request hacia `develop`.

---

## 1. Concepto general (importante leer)

El proyecto se organiza en **tres repositorios separados**:

* `psira-docker` → infraestructura (Docker, base de datos, proxy, etc.)
* `psira-frontend` → aplicación frontend
* `psira-backend` → aplicación backend

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
```

⚠️ **Importante:**
Los nombres y la ubicación relativa importan porque `docker-compose.dev.yml` asume esta estructura.

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

## 7. Flujo de trabajo con Git (antes de empezar a editar)

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
# Creas tu propia rama
git checkout -b nombre/descripcion-corta
```

Ejemplo:

```bash
git checkout -b juan/fix-questionnaire-error
```

Una vez en la rama propia se pueden realizar todos los cambios que uno quiera.

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

## 10. Pull Requests

Cuando el cambio esté listo:

```bash
git push -u origin nombre/descripcion-corta
```

Luego:

1. Abrir Pull Request en GitHub
2. Base: `develop`
3. Describir:

   * qué se cambió
   * cómo probarlo
   * si rompe algo conocido

⚠️ Solo el responsable del proyecto mergea a `develop` o `main`.

### ❗ Una vez se resuelve el push a develop ELIMINAR rama de trabajo y crear una nueva para nuevos trabajos
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
