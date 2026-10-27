---
id: "docker-containers-and-deploy"
version: "1.0.0"
domain: "DevOps"
technologies:
  - "Docker"
  - "Docker Compose"
  - "Multi-Stage Builds"
  - "Nginx Alpine"
  - "Dual-Repo Strategy"
triggers:
  - "Containerizar una aplicación backend (Python/Django/Node) o frontend (React/Vite)"
  - "Optimizar el tamaño y la seguridad de imágenes Docker para producción"
  - "Configurar un entorno local con Docker Compose para desarrollo"
  - "Desplegar SPAs estáticas en GitHub Pages protegiendo el código fuente privado (Estrategia Dual-Repo)"
related_skills:
  - "gcp-core-services"
  - "easypanel-deployment"
  - "security-hardening"
source_generation: "core"
---

# Docker Containers & Despliegue — Multi-Stage Builds y Estrategia Dual-Repo

> **TL;DR para IA:** Aplica esta skill para crear imágenes Docker seguras, mínimas y multi-stage sin privilegios root, configurar Compose reproducible y ejecutar despliegues de frontend estático desacoplados mediante doble repositorio.

---

## 1. Antipatrones Críticos en Docker

### ❌ Antipatrón 1: Ejecutar contenedores como usuario `root`
Permitir que el proceso principal corra como root expone el host del servidor ante fugas o exploits de escape de contenedor.

### ❌ Antipatrón 2: Dejar herramientas de compilación en la imagen final
Instalar `gcc`, `build-essential`, `python3-dev` en la misma capa que se ejecuta en producción infla la imagen a más de 1 GB y añade cientos de CVEs innecesarios.

### ❌ Antipatrón 3: Olvidar `.dockerignore`
Copiar `node_modules`, `.env`, `.git` o logs locales dentro de la imagen hornea credenciales y hace los builds extremadamente lentos.

---

## 2. Reglas de Oro

1. **Multi-Stage Builds Obligatorios**: Separar la fase de compilación/instalación (`builder`) de la fase de ejecución ligera (`runner`/`final`).
2. **Usuario No-Root**: Siempre declarar un usuario dedicado (`USER appuser`) antes del `CMD`/`ENTRYPOINT`.
3. **Imágenes Base Específicas**: Fijar versiones completas de runtimes slim o alpine (ej. `python:3.11-slim-bookworm` o `node:20-alpine3.19`), nunca `:latest`.
4. **Orden de Capas por Frecuencia de Cambio**: Copiar primero archivos de definición de dependencias (`package.json`, `pyproject.toml`) antes del código fuente para maximizar la caché de capas.

---

## 3. Dockerfile Multi-Stage: Python Backend (Producción)

```dockerfile
# --- STAGE 1: Builder ---
FROM python:3.11-slim-bookworm AS builder

WORKDIR /app

RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    libpq-dev \
    && rm -rf /var/lib/apt/lists/*

COPY pyproject.toml .
RUN pip install --no-cache-dir --user -e .

# --- STAGE 2: Runtime Mínimo ---
FROM python:3.11-slim-bookworm AS final

WORKDIR /app

RUN apt-get update && apt-get install -y --no-install-recommends \
    libpq5 \
    curl \
    && rm -rf /var/lib/apt/lists/*

# Crear usuario sin privilegios
RUN groupadd -r appgroup && useradd -r -g appgroup -u 10001 appuser

# Copiar paquetes instalados desde el builder
COPY --from=builder /root/.local /home/appuser/.local
COPY --chown=appuser:appgroup src/ /app/src/

ENV PATH=/home/appuser/.local/bin:$PATH \
    PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1

USER appuser

EXPOSE 8000

CMD ["gunicorn", "src.main:app", "--bind", "0.0.0.0:8000", "--workers", "3"]
```

---

## 4. Dockerfile Multi-Stage: React / Vite Frontend (Producción)

```dockerfile
# --- STAGE 1: Build de Artefactos Estáticos ---
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# --- STAGE 2: Servidor Web Nginx Alpine ---
FROM nginx:1.25-alpine
COPY --from=builder /app/dist /usr/share/nginx/html
# Configuración SPA (evita error 404 en deep links redirigiendo a index.html)
RUN echo 'server { \
    listen 80; \
    location / { \
        root /usr/share/nginx/html; \
        index index.html; \
        try_files $uri $uri/ /index.html; \
    } \
}' > /etc/nginx/conf.d/default.conf

EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

---

## 5. Estrategia de Despliegue Dual-Repo (SPA estática)

> Consolidado y abstraído de `Skills_g1/dual-repo-static-spa-deployment` y `Skills_g2/dual-repo-github-pages-deploy`

Cuando se requiere publicar una demo en hosting estático público (como GitHub Pages) manteniendo el código fuente en un repositorio privado:

```
[ Repo Privado (Fuente) ]  ──npm run build──▶  [ Directorio dist/ ]
                                                       │
                                              Script de sincronización
                                                       │
                                                       ▼
                                            [ Repo Público (Deploy) ]
                                            (Solo HTML/JS/CSS compilado)
```

### Script de Despliegue Automatizado (`deploy-public.sh`)

```bash
#!/usr/bin/env bash
set -e

# CONFIG: URL del repositorio público de distribución
PUBLIC_REPO_URL="git@github.com:organizacion/demo-publica.git"
DIST_DIR="dist"

echo "1. Construyendo artefactos minimizados..."
npm run build

echo "2. Preparando deep links y Jekyll bypass..."
cp "$DIST_DIR/index.html" "$DIST_DIR/404.html"
touch "$DIST_DIR/.nojekyll"

echo "3. Inicializando y publicando hacia el repo público..."
cd "$DIST_DIR"
git init
git checkout -B gh-pages
git add -A
git commit -m "Deploy automated build $(date -u +'%Y-%m-%dT%H:%M:%SZ')"
git push -f "$PUBLIC_REPO_URL" gh-pages:gh-pages

echo "✅ Despliegue completado sin exponer código fuente privado."
```

---

## 6. Checklist de Verificación

- [ ] Archivo `.dockerignore` configurado en la raíz con `.git`, `.env*`, `node_modules`, `__pycache__`
- [ ] La imagen final corre bajo un usuario sin privilegios root (`USER appuser`)
- [ ] Ningún secreto ni clave de API está presente en sentencias `ENV` o `ARG` del Dockerfile
- [ ] El contenedor responde a un endpoint de *healthcheck* (`/health` o `/ready`)
- [ ] Para SPAs estáticas en Nginx, la configuración maneja correctamente el fallback a `/index.html`
