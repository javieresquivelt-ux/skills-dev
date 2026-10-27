---
id: "frontend-core-patterns"
version: "1.0.0"
domain: "Frontend"
technologies:
  - "JavaScript (ESNext)"
  - "TypeScript"
  - "CSS Custom Properties / Sass"
  - "Local-First Storage"
  - "RBAC Guard"
  - "Google Sheets / Apps Script (GAS)"
triggers:
  - "Crear una UI que adapte colores, logos o textos en runtime desde una API o Sheets sin recompilar"
  - "Implementar control de acceso por roles (RBAC) y rutas protegidas en una SPA"
  - "Desarrollar aplicaciones offline-first con datos mock locales y sincronización remota"
  - "Integrar Google Sheets como base de datos o Google Apps Script como API backend"
  - "Manejar zonas horarias de negocio (Wallclock) y agendas sin desfase de fechas"
related_skills:
  - "react-component-patterns"
  - "security-hardening"
source_generation: "core"
---

# Frontend Core — UI Dinámica, RBAC Modular, Local-First y Serverless BaaS

> **TL;DR para IA:** Esta skill unifica los patrones frontend de alto rendimiento: theming dinámico por configuración externa, control modular de roles (RBAC), arquitectura de almacenamiento híbrido (Local-First) e integración con Google Apps Script / Google Sheets como backend serverless sin costo.

---

## 1. Módulo: Config-Driven Dynamic UI & Theming

Permite que una aplicación web altere su identidad visual (colores primarios, logotipo, fuentes, copys) en tiempo de ejecución leyendo un JSON o un endpoint sin necesidad de recompilar la aplicación.

### Implementación mediante Tokens CSS en Runtime

```typescript
export interface AppBrandConfig {
  primaryColor: string;
  secondaryColor: string;
  surfaceColor: string;
  textColor: string;
}

export function aplicarThemingDinamico(config: AppBrandConfig): void {
  const root = document.documentElement;
  root.style.setProperty("--brand-primary", config.primaryColor);
  root.style.setProperty("--brand-secondary", config.secondaryColor);
  root.style.setProperty("--surface-bg", config.surfaceColor);
  root.style.setProperty("--text-main", config.textColor);
}
```

```css
/* Consumo semántico en CSS/SCSS */
:root {
  --brand-primary: #0052cc;
  --brand-secondary: #0747a6;
  --surface-bg: #ffffff;
  --text-main: #172b4d;
}

.btn-primary {
  background-color: var(--brand-primary);
  color: #ffffff;
  transition: filter 0.2s ease;
}
.btn-primary:hover {
  filter: brightness(0.9);
}
```

---

## 2. Módulo: Modular RBAC en SPAs (Doble Gate de Seguridad)

> Consolidado de `Skills_g2/modular-rbac-spa`

### Principio: Desacoplar Roles de Módulos
Nunca valides roles crudos dentro de las pantallas (`if (user.role === 'admin')`). Define **capacidades o módulos permitidos** que el rol adquiere:

```typescript
export type RolUsuario = "SUPER_ADMIN" | "OPERADOR" | "LECTOR";

export interface ModuloPermitido {
  id: string;
  ruta: string;
  etiqueta: string;
  icono: string;
}

const MATRIZ_ACCESO: Record<RolUsuario, string[]> = {
  SUPER_ADMIN: ["dashboard", "ventas", "usuarios", "configuracion"],
  OPERADOR: ["dashboard", "ventas"],
  LECTOR: ["dashboard"],
};

export function tieneAccesoAModulo(rol: RolUsuario, moduloId: string): boolean {
  return MATRIZ_ACCESO[rol]?.includes(moduloId) ?? false;
}
```

### Doble Gate:
1. **Gate Visual (Sidebar/Navegación)**: Filtra los enlaces para no renderizar botones inaccesibles.
2. **Gate de Enrutamiento (Router Guard)**: Bloquea el acceso a la ruta si el usuario tipea la URL manualmente en la barra de navegación, redirigiendo a `/403`.

---

## 3. Módulo: Persistencia Local-First & Patrón Repository

> Consolidado de `Skills_g2/local-first-hybrid-storage`

La aplicación debe cargar al instante usando datos locales (mock / `localStorage` / `IndexedDB`) y actualizarse en segundo plano si hay conectividad con la API.

```typescript
export interface RepositorioDatos<T> {
  listar(): Promise<T[]>;
  guardar(item: T): Promise<T>;
}

export class RepositorioHibrido<T extends { id: string }> implements RepositorioDatos<T> {
  constructor(
    private readonly storageKey: string,
    private readonly semillas: T[],
    private readonly apiUrl?: string
  ) {}

  async listar(): Promise<T[]> {
    const local = localStorage.getItem(this.storageKey);
    let datosLocales: T[] = local ? JSON.parse(local) : this.semillas;

    // Sincronización en segundo plano (si hay API disponible)
    if (this.apiUrl && navigator.onLine) {
      try {
        const resp = await fetch(this.apiUrl);
        if (resp.ok) {
          const remotos: T[] = await resp.json();
          localStorage.setItem(this.storageKey, JSON.stringify(remotos));
          return remotos;
        }
      } catch {
        // Fallback transparente al almacenamiento local
      }
    }
    return datosLocales;
  }

  async guardar(item: T): Promise<T> {
    const items = await this.listar();
    const nuevos = [...items.filter(i => i.id !== item.id), item];
    localStorage.setItem(this.storageKey, JSON.stringify(nuevos));

    if (this.apiUrl && navigator.onLine) {
      await fetch(this.apiUrl, {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify(item),
      }).catch(() => {/* Encolar para sincronización offline posterior */});
    }
    return item;
  }
}
```

---

## 4. Módulo: Google Sheets & Apps Script como BaaS Serverless

> Consolidado de `Skills_g1/sheets-as-database`, `Skills_g1/gas-rest-api-design` y `Skills_g2/google-sheets-baas`

### Antipatrón Crítico de Tipado en Google Sheets
En Google Sheets / Apps Script, el valor de texto `"false"` al enviarse en un payload se evalúa como `truthy` en JavaScript si no se sanitiza:
```javascript
// ❌ MAL
const activo = Boolean(row[3]); // "false" string es TRUE en JS

// ✅ BIEN: Normalización estricta de booleanos
function parseBoolean(val) {
  if (typeof val === "boolean") return val;
  if (typeof val === "string") {
    const s = val.trim().toLowerCase();
    return s === "true" || s === "1" || s === "si" || s === "yes";
  }
  return Boolean(val);
}
```

### Manejo de Concurrencia con `LockService` (GAS)
```javascript
function doPost(e) {
  const lock = LockService.getScriptLock();
  // Esperar hasta 10 segundos por el cerrojo para evitar colisiones
  if (!lock.tryLock(10000)) {
    return ContentService.createTextOutput(
      JSON.stringify({ error: "Servidor ocupado. Reintente en unos segundos." })
    ).setMimeType(ContentService.MimeType.JSON);
  }

  try {
    const payload = JSON.parse(e.postData.contents);
    const sheet = SpreadsheetApp.getActiveSpreadsheet().getSheetByName("Datos");
    sheet.appendRow([payload.id, payload.nombre, new Date().toISOString()]);
    
    return ContentService.createTextOutput(
      JSON.stringify({ status: "OK" })
    ).setMimeType(ContentService.MimeType.JSON);
  } finally {
    lock.releaseLock();
  }
}
```

---

## 5. Módulo: Wallclock Engine — Gestión de Zonas Horarias Comerciales

> Consolidado de `Skills_g1/timezone-wallclock-engine`

Para sistemas de reservas, citas o turnos de negocio: **nunca almacenes turnos como objetos `Date` locales del navegador del usuario**.

* **La Regla del Reloj de Pared**: Una cita pactada a las "10:00 AM en Santiago de Chile" debe operar en la zona horaria del negocio (`America/Santiago`), independientemente de si el cliente navega desde Londres o Tokio.
* **Modelo en Minutos Enteros**:
  ```typescript
  // Almacenar horas del negocio como minutos desde medianoche:
  // 10:00 AM = 10 * 60 = 600 minutos
  export function horaAMinutos(horaStr: string): number {
    const [hh, mm] = horaStr.split(":").map(Number);
    return hh * 60 + mm;
  }
  ```
  Esto previene bugs de desfase por Horario de Verano (DST) y desfases de husos horarios.

---

## 6. Checklist de Verificación

- [ ] Las variables visuales se controlan mediante tokens CSS nativos (`var(--...)`) aplicables en runtime
- [ ] El RBAC valida capacidades/módulos y cuenta con doble barrera (ocultamiento visual + guardia en el router)
- [ ] La aplicación puede renderizar pantallas iniciales a partir de datos mock locales sin requerir red activa
- [ ] En integraciones con Google Sheets, los booleanos y fechas cuentan con serializadores defensivos explícitos
- [ ] Las operaciones críticas de escritura en Google Apps Script están protegidas mediante `LockService.getScriptLock()`
