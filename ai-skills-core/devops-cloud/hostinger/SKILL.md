---
id: "hostinger-dns-setup"
version: "1.0.0"
domain: "DevOps"
technologies:
  - "Hostinger DNS Zone"
  - "DNS Records (A, CNAME, TXT, MX)"
  - "SPF / DKIM / DMARC"
  - "Cloud Run / VPS Mapping"
triggers:
  - "Configurar un nuevo dominio o subdominio en Hostinger"
  - "Apuntar un dominio hacia un VPS (Easypanel) o Cloud Run / GitHub Pages"
  - "Configurar registros de verificación de Google Search Console o Google Workspace"
  - "Solucionar fallos de resolución de nombres o propagación de DNS"
related_skills:
  - "easypanel-deployment"
  - "gcp-core-services"
source_generation: "core"
---

# Hostinger — Configuración de Registros DNS, Dominios y Enrutamiento Cloud

> **TL;DR para IA:** Usa esta skill para gestionar la Zona DNS de Hostinger, configurar apuntamientos limpios (A, CNAME), integrar registros de autenticación de correos (SPF/DKIM/DMARC) y enlazar servidores VPS o servicios en la nube.

---

## 1. Tipos de Registros DNS Esenciales

| Tipo | Propósito | Ejemplo de Uso |
|---|---|---|
| **A** | Asocia un nombre de host a una dirección IPv4 directa | `api.tudominio.com` $\to$ `185.192.12.34` (VPS / Easypanel) |
| **AAAA** | Asocia un nombre de host a una dirección IPv6 directa | `tudominio.com` $\to$ `2001:0db8:...` |
| **CNAME** | Alias de un nombre de dominio hacia otro dominio canónico | `www.tudominio.com` $\to$ `tudominio.com` o `ghs.googlehosted.com` |
| **TXT** | Registros de texto para validación de propiedad y seguridad | Verificación Google, SPF (`v=spf1 ...`), DMARC |
| **MX** | Servidores de correo entrante para el dominio | Registros de Google Workspace o Titan Mail |

---

## 2. Antipatrones a Evitar

### ❌ Antipatrón 1: CNAME en el dominio raíz (`@` / apex)
Definir un registro CNAME en el dominio raíz (`tudominio.com`) viola la especificación RFC 1034 y rompe los registros MX (correos) del dominio.
**Regla**: El dominio raíz (`@`) siempre debe usar registros **A** (o ALIAS / ANAME si el proveedor lo soporta).

### ❌ Antipatrón 2: TTLs excesivamente altos durante migraciones
Configurar un TTL de 86400 segundos (24 horas) antes de una migración de servidor significa que cualquier error de IP tardará un día entero en propagarse para corregirse.
**Regla**: Bajar el TTL a 300 segundos (5 minutos) 48 horas antes de cualquier cambio de infraestructura.

### ❌ Antipatrón 3: Múltiples registros SPF contradictorios
Tener más de un registro `v=spf1` en la zona TXT invalida automáticamente la autenticación de correos y causa que los emails caigan en SPAM. Debe existir un único registro SPF consolidado con múltiples `include:`.

---

## 3. Matriz de Configuración por Destino

### Caso A: Apuntar hacia Servidor VPS / Easypanel
```text
Tipo    Nombre      Contenido (Valor)       TTL
A       @           <IP_PUBLICA_DEL_VPS>    300 (o 14400)
A       api         <IP_PUBLICA_DEL_VPS>    300 (o 14400)
A       n8n         <IP_PUBLICA_DEL_VPS>    300 (o 14400)
CNAME   www         tudominio.com           14400
```

### Caso B: Apuntar hacia GitHub Pages (Custom Domain)
```text
Tipo    Nombre      Contenido (Valor)       TTL
A       @           185.199.108.153         14400
A       @           185.199.109.153         14400
A       @           185.199.110.153         14400
A       @           185.199.111.153         14400
CNAME   demo        <usuario>.github.io     14400
```

### Caso C: Integración con Google Workspace y Verificación
```text
Tipo    Nombre      Contenido (Valor)                                           TTL
TXT     @           google-site-verification=XXXXXXXXXXXXXXXXXXXX               3600
TXT     @           v=spf1 include:_spf.google.com ~all                        3600
MX      @           1 ASPMX.L.GOOGLE.COM                                        3600
MX      @           5 ALT1.ASPMX.L.GOOGLE.COM                                   3600
```

---

## 4. Procedimiento de Diagnóstico y Verificación

Antes de declarar un despliegue como operativo, verificar la resolución DNS desde la terminal:

```bash
# Consultar registro A del dominio
dig +short A api.tudominio.com

# Consultar registros TXT (SPF y validaciones)
dig +short TXT tudominio.com

# Verificar propagación global inmediata contra Cloudflare y Google DNS
nslookup api.tudominio.com 1.1.1.1
nslookup api.tudominio.com 8.8.8.8
```

---

## 5. Checklist de Verificación

- [ ] El dominio raíz (`@`) no tiene asignado un registro CNAME
- [ ] No existen registros `A` residuales o duplicados apuntando a servidores antiguos
- [ ] Existe un único registro TXT que comienza con `v=spf1`
- [ ] El TTL se ajustó adecuadamente (300s en pruebas/migraciones, 14400s en producción estable)
- [ ] Los subdominios de servicios (`api`, `app`, `n8n`) resuelven a la IP correcta vía `dig`
