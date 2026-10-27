---
id: "sass-7-1-architecture"
version: "1.0.0"
domain: "Frontend"
technologies:
  - "Sass / SCSS"
  - "CSS Custom Properties"
  - "Flexbox / Grid"
  - "WCAG 2.5.5"
triggers:
  - "Diseñar o maquetar la arquitectura de estilos CSS de una aplicación web"
  - "Implementar Dark/Light mode con tokens semánticos en CSS"
  - "Evitar compresión o distorsión de elementos en Flexbox (flex-shrink: 0)"
  - "Refactorizar estilos CSS desordenados hacia el patrón modular Sass 7-1"
related_skills:
  - "react-component-patterns"
  - "frontend-core-patterns"
source_generation: "g2"
---

# Arquitectura CSS Profesional con Patrón Sass 7-1 y Diseño de Tokens

Esta skill proporciona una guía integral para estructurar los estilos de una aplicación web moderna utilizando el patrón arquitectónico **Sass 7-1**, complementado con **Tokens de Diseño en Custom Properties CSS** para temas claro/oscuro, blindaje estructural contra distorsión Flexbox y cumplimiento de accesibilidad táctil móvil (WCAG).

---

## 1. Estructura Canónica de Directorios (Patrón 7-1)

El patrón 7-1 organiza los estilos en 7 carpetas temáticas con un único manifiesto compilador (`main.scss`). Cada módulo tiene un rol estricto y predecible:

```text
src/scss/
├── abstracts/        # Variables Sass, funciones matemáticas y mixins globales (sin salida CSS)
│   ├── _variables.scss
│   ├── _functions.scss
│   └── _mixins.scss
├── base/             # Reset, normalización de estilos y tipografía global
│   ├── _reset.scss
│   ├── _typography.scss
│   └── _base.scss
├── themes/           # Tokens semánticos y paletas de color en CSS Custom Properties
│   ├── _theme-dark.scss
│   └── _theme-light.scss
├── components/       # Elementos visuales reutilizables e independientes del contexto
│   ├── _buttons.scss
│   ├── _cards.scss
│   ├── _badges.scss
│   ├── _forms.scss
│   └── _modals.scss
├── layout/           # Estructura del cascarón de la aplicación (Shell)
│   ├── _grid.scss
│   ├── _header.scss
│   └── _sidebar.scss
├── pages/            # Estilos específicos de vistas complejas
│   ├── _dashboard.scss
│   └── _auth.scss
├── vendors/          # Estilos de librerías de terceros (si aplica)
└── main.scss         # Manifiesto central que orquesta las importaciones (@use)
```

---

## 2. Sistema de Tokens Semánticos (Dark & Light Mode)

### Principio de Diseño
No definas colores estáticos en los componentes. Utiliza **Custom Properties CSS** contextualizadas en la raíz (`:root` o `[data-theme]`). El componente únicamente consume `var(--color-nombre)`, lo que permite alternar temas instantáneamente sin re-renderizados de JavaScript ni recálculo de estilos.

### Implementación Canónica

**1. Definición en `themes/_theme-light.scss`:**
```scss
[data-theme='light'] {
  --color-bg-app: #f8fafc;
  --color-bg-surface: #ffffff;
  --color-bg-surface-elevated: #ffffff;
  --color-border: #e2e8f0;
  --color-text-main: #0f172a;
  --color-text-secondary: #475569;
  --color-text-muted: #94a3b8;
  --color-primary: #2563eb;
  --color-primary-subtle: rgba(37, 99, 235, 0.1);
  --color-success: #16a34a;
  --color-danger: #dc2626;
  --color-warning: #d97706;
}
```

**2. Definición en `themes/_theme-dark.scss`:**
```scss
[data-theme='dark'] {
  --color-bg-app: #0b0f19;
  --color-bg-surface: #111827;
  --color-bg-surface-elevated: #1f2937;
  --color-border: #374151;
  --color-text-main: #f9fafb;
  --color-text-secondary: #d1d5db;
  --color-text-muted: #6b7280;
  --color-primary: #3b82f6;
  --color-primary-subtle: rgba(59, 130, 246, 0.15);
  --color-success: #22c55e;
  --color-danger: #ef4444;
  --color-warning: #f59e0b;
}
```

---

## 3. Blindaje contra Compresión Flexbox en Layouts

### El Problema de la Compresión Indeseada
En interfaces con barra lateral (Sidebar) y área principal (Main Content), el algoritmo de Flexbox por defecto puede comprimir el sidebar (`flex-shrink: 1`) cuando el contenido del área principal desborda o cuando los enlaces del menú tienen textos extensos que quiebran línea de forma irregular, dando la impresión de que el sidebar "cambia de ancho" al navegar entre páginas.

### Solución Arquitectónica
Aplica un **blindaje en tres niveles**:

```scss
// layout/_sidebar.scss
.app-sidebar {
  width: 260px;
  min-width: 260px;      // 1. Blindaje contra reducción Flexbox
  flex-shrink: 0;        // 2. Prohíbe explícitamente el encogimiento
  overflow: hidden;      // 3. Previene que desbordes internos fuercen cambio de dimensiones
  
  &__link {
    display: flex;
    align-items: center;
    width: 100%;
    white-space: nowrap; // 4. Impide que textos largos quiebren en 2 líneas
    overflow: hidden;
    text-overflow: ellipsis;
  }
}
```

---

## 4. Accesibilidad Táctil en Vistas Móviles (WCAG 2.5.5)

### Regla del Target Táctil Mínimo
En pantallas táctiles (móvil y tablet), los elementos interactivos críticos (botones de menú hamburguesa, botones de cierre de modal o links de navegación) deben tener un área mínima de interacción de **44 × 44 píxeles**:

```scss
// layout/_header.scss
.app-header__menu-btn {
  display: flex;
  align-items: center;
  justify-content: center;
  min-width: 44px;
  min-height: 44px;
  padding: 0;
  border-radius: var(--radius-md);
  
  @include mq('lg') {
    display: none; // Oculto en pantallas grandes
  }
}
```

---

## 5. Cuadrículas Elásticas sin Utilitarios Inline

Para evitar dejar huecos vacíos cuando la visibilidad de elementos se altera por permisos o datos condicionales, define clases elásticas en `layout/_grid.scss`:

```scss
// layout/_grid.scss
.grid {
  display: grid;
  gap: var(--spacing-md);

  &--1 { grid-template-columns: 1fr; }
  &--2 { grid-template-columns: repeat(auto-fit, minmax(280px, 1fr)); }
  &--3 { grid-template-columns: repeat(auto-fit, minmax(240px, 1fr)); }
  &--4 { grid-template-columns: repeat(auto-fit, minmax(200px, 1fr)); }
}

.grid-col-span-full {
  grid-column: 1 / -1;
}
```

---

## 6. Anti-Patrones y Trampas Comunes (Gotchas)

1. ❌ **Estilos Inline (`style={{ ... }}`) en componentes**:
   * *Problema*: Dificulta el mantenimiento, rompe el sistema de temas y evade la compilación de Sass.
   * *Solución*: Crear clases atómicas o modificadores BEM en el archivo SCSS respectivo (ej. `.badge-group`, `.form-check`).
2. ❌ **Anidación excesiva en Sass (> 3 niveles)**:
   * *Problema*: Genera selectores con altísima especificidad difíciles de sobrescribir (ej. `.app .sidebar ul li a span`).
   * *Solución*: Seguir la metodología BEM estricta (`.block__element--modifier`).
3. ❌ **Colores Hexadecimales dispersos en componentes**:
   * *Problema*: Al cambiar de tema claro a oscuro, los colores se quedan congelados.
   * *Solución*: Usar siempre variables CSS semánticas (`var(--color-...)`).

---

## 7. Checklist de Verificación de Calidad

- [ ] Toda regla CSS reside dentro de un archivo parcial de Sass (`_*.scss`).
- [ ] No existen propiedades `style={...}` en el código JSX/HTML.
- [ ] El cambio de atributo `data-theme="dark"` / `data-theme="light"` en el `<html>` cambia todos los contrastes sin parpadeos.
- [ ] El sidebar mantiene exactamente su ancho en todas las vistas de la aplicación.
- [ ] Todos los botones de acción rápida en móvil cuentan con un área táctil de al menos 44×44px.
- [ ] `npm run build` compila los estilos en menos de 1 segundo sin advertencias de deprecación.
