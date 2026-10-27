---
id: "easypanel-deployment"
version: "1.0.0"
domain: "DevOps"
technologies:
  - "Easypanel"
  - "Docker / Traefik"
  - "VPS (Ubuntu / Debian)"
  - "Let's Encrypt SSL"
triggers:
  - "Desplegar aplicaciones o bases de datos en un servidor VPS usando Easypanel"
  - "Configurar variables de entorno y volúmenes persistentes en Easypanel"
  - "Vincular dominios y subdominios con certificados SSL automáticos en Easypanel"
  - "Configurar servicios auxiliares (PostgreSQL, Redis, n8n) en Easypanel"
related_skills:
  - "docker-containers-and-deploy"
  - "hostinger-dns-setup"
  - "security-hardening"
source_generation: "core"
---

# Easypanel — Despliegue de Aplicaciones y Servicios en Servidores VPS

> **TL;DR para IA:** Usa esta skill para estructurar y automatizar el despliegue de aplicaciones Docker, bases de datos y servicios (como n8n) sobre un VPS gestionado con Easypanel, asegurando persistencia de datos y certificados SSL sin fricción.

---

## 1. Arquitectura de Despliegue con Easypanel

Easypanel actúa como una capa de orquestación ligera sobre Docker y Traefik en tu propio servidor VPS:

```
Internet (HTTPS)
   │
   ▼
[ Traefik Reverse Proxy (Puerto 80/443 + Let's Encrypt automático) ]
   │
   ├── api.tudominio.com   ──▶ [ Contenedor Backend (Django/Python) ]
   ├── app.tudominio.com   ──▶ [ Contenedor Frontend (SPA Nginx) ]
   └── n8n.tudominio.com   ──▶ [ Contenedor n8n + Volumen Persistente ]
```

---

## 2. Antipatrones a Evitar

### ❌ Antipatrón 1: Almacenar bases de datos o uploads sin volúmenes persistentes
Si despliegas SQLite, PostgreSQL o subidas de archivos en la capa efímera del contenedor, cada redeploy o actualización destruirá todos los datos.

### ❌ Antipatrón 2: Exponer puertos de bases de datos al internet público
Publicar el puerto `5432` o `6379` al exterior sin reglas de firewall invita a ataques de fuerza bruta continuos. Las bases de datos deben comunicarse mediante la red interna de Easypanel.

### ❌ Antipatrón 3: Ignorar límites de recursos en VPS compartidos
Un contenedor con fuga de memoria puede congelar todo el servidor VPS y tumbar los demás servicios si no se le configuran topes de memoria.

---

## 3. Reglas de Oro

1. **Volúmenes Persistentes para Todo Estado**: Declarar montajes (`/data`, `/app/uploads`) vinculados al almacenamiento del host para cualquier dato que deba sobrevivir a reinicios.
2. **Red Interna para Servicios**: Conectar el backend con PostgreSQL o Redis usando sus nombres de servicio internos (ej. `postgres:5432`), manteniendo los puertos cerrados hacia el internet.
3. **Límites de Memoria Preventivos**: Asignar topes de memoria (ej. `512MB` para un API pequeño, `1GB` para n8n) en la configuración de la app en Easypanel.
4. **Healthchecks Activos**: Configurar un endpoint de comprobación (`/health`) para que Easypanel reintente el despliegue con zero-downtime.

---

## 4. Configuración Estándar de una App en Easypanel

### Especificación de Servicio (Project Schema)

| Parámetro | Valor Recomendado | Propósito |
|---|---|---|
| **Build Type** | `Dockerfile` | Permite optimizaciones multi-stage personalizadas |
| **Dockerfile Path** | `./Dockerfile` | Ubicación en el repositorio de código |
| **Port** | `8000` (Backend) / `80` (Frontend) | Puerto interno donde escucha el proceso |
| **Domain** | `api.ejemplo.com` | Traefik emite certificado SSL automáticamente |
| **Memory Limit** | `512 MB` - `1024 MB` | Evita saturación global de la memoria RAM del VPS |
| **Restart Policy** | `always` | Reinicia el contenedor si el servidor se reinicia |

### Variables de Entorno Clave
```ini
# CONFIG: Ajustar valores en la sección "Environment" de Easypanel
NODE_ENV=production
PORT=8000
DATABASE_URL=postgres://usuario:password@postgres:5432/mibasedatos
CORS_ALLOWED_ORIGINS=https://app.ejemplo.com
```

---

## 5. Despliegue de n8n en Easypanel

Para levantar una instancia productiva y persistente de n8n:

1. **Crear Servicio**: Añadir servicio tipo `App` usando la imagen oficial `docker.n8n.io/n8nio/n8n:latest`.
2. **Volumen Persistente**:
   * Mount Path: `/home/node/.n8n`
3. **Variables de Entorno Mínimas**:
   ```ini
   N8N_HOST=n8n.ejemplo.com
   N8N_PORT=5678
   N8N_PROTOCOL=https
   WEBHOOK_URL=https://n8n.ejemplo.com/
   GENERIC_TIMEZONE=America/Santiago
   EXECUTIONS_DATA_PRUNE=true
   EXECUTIONS_DATA_MAX_AGE=168
   ```
4. **Dominio**: Asignar `n8n.ejemplo.com` (apuntando previamente el DNS en Hostinger/Cloudflare al IP del VPS).

---

## 6. Checklist de Verificación

- [ ] El registro DNS tipo `A` del dominio apunta a la IP pública del servidor VPS antes de añadirlo en Easypanel
- [ ] El certificado SSL se emitió correctamente por Let's Encrypt (status HTTPS verde en el navegador)
- [ ] Todo directorio con escrituras en disco cuenta con un volumen persistente configurado
- [ ] La base de datos es inaccesible desde el exterior (puertos públicos desactivados)
- [ ] Se probó un reinicio del contenedor verificando que la data persiste intacta
