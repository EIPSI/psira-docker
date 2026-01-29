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

1. **Docker Desktop**

   * Activar *Use WSL 2 based engine*
2. **WSL2 con Ubuntu**
3. **Git**
4. **VS Code** (recomendado)

   * Extensión *WSL* (opcional pero muy útil)

Verificar instalación:

```bash
docker version
docker compose version
```

---

## 3. Estructura de carpetas esperada

En tu computadora (preferentemente dentro de WSL), crear una carpeta común, por ejemplo:

```text
~/eipsi/
  psira-docker/
  psira-frontend/
  psira-backend/
```

Clonar los repositorios en esa estructura:

```bash
cd ~/eipsi
git clone -b develop <repo-psira-docke>
git clone -b develop <repo-psira-frontend>
git clone -b develop <repo-psira-backend>
```

⚠️ **Importante:**
Los nombres y la ubicación relativa importan porque `docker-compose.dev.yml` asume esta estructura.

---

## 4. Variables de entorno (solo una vez)

Dentro del repo **`psira-docker`**:

```bash
cd psira-docker
cp .env.local.example .env
```

El archivo `.env` **NO debe subirse al repositorio**.
Es solo para uso local.

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

---

## 6. Accesos locales

Una vez levantado el entorno:

* **Frontend:**
  [http://localhost:8080](http://localhost:8080)

* **GraphQL (backend):**
  [http://localhost:8080/graphql](http://localhost:8080/graphql)

* **Base de datos:**
  No es necesario acceder directamente para desarrollo habitual.

---

## 7. Flujo de trabajo con Git

### Ramas

* `main` → producción (no tocar)
* `develop` → integración/staging
* `nombre/tarea` → ramas de trabajo individual

### Crear una rama de trabajo

```bash
git checkout develop
git pull origin develop
git checkout -b nombre/descripcion-corta
```

Ejemplo:

```bash
git checkout -b juan/fix-questionnaire-error
```

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
  up -d --build
```

4. refrescar el navegador

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
