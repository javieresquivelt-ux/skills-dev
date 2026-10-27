---
id: "react-component-patterns"
version: "1.0.0"
domain: "Frontend"
technologies:
  - "React 18+"
  - "TypeScript"
  - "React Hooks"
  - "Context API"
  - "Vite"
triggers:
  - "Crear un componente React complejo con estado"
  - "Manejar estado global en una SPA React"
  - "Optimizar renders innecesarios en componentes"
  - "Estructurar una nueva aplicación React desde cero"
  - "Implementar Custom Hooks para lógica reutilizable"
related_skills:
  - "sass-7-1-architecture"
  - "modular-rbac-spa"
  - "local-first-hybrid-storage"
  - "error-handling-patterns"
source_generation: "core"
---

# React 18+ — Hooks, Context, Patrones de Componentes y Optimización

> **TL;DR para IA:** Carga esta skill para estructurar componentes React correctamente,
> elegir el patrón de estado adecuado y evitar renders innecesarios.

---

## 1. Estructura de Proyecto React + Vite

```
src/
├── main.tsx              ← Entry point, providers globales
├── App.tsx               ← Router principal
├── assets/               ← Imágenes, fuentes estáticas
├── styles/               ← Ver skill sass-7-1-architecture
│   └── main.scss
├── components/           ← Componentes reutilizables (sin lógica de negocio)
│   ├── ui/               ← Primitivos: Button, Input, Modal, Card
│   └── layout/           ← Header, Sidebar, PageWrapper
├── features/             ← Feature modules (estado + UI del dominio)
│   └── <feature>/
│       ├── components/   ← Componentes específicos del feature
│       ├── hooks/        ← Custom hooks del feature
│       ├── context/      ← Context API del feature
│       └── types.ts      ← Tipos TypeScript del feature
├── hooks/                ← Custom hooks globales
├── services/             ← Llamadas a APIs externas
│   └── api/
│       ├── client.ts     ← Configuración del cliente HTTP
│       └── <recurso>.ts
├── store/                ← Estado global (Context API o Zustand)
└── types/                ← Tipos TypeScript globales
```

---

## 2. Antipatrones a Evitar

### ❌ Antipatrón 1: Estado sin necesidad en componentes hoja
```tsx
// ❌ MAL — Componente hoja con estado innecesario
function BotonGuardar({ datos }: Props) {
  const [guardado, setGuardado] = useState(false);  // ← ¿Por qué el botón sabe del estado de guardado?
  return <button onClick={() => setGuardado(true)}>Guardar</button>;
}
```

### ❌ Antipatrón 2: Efectos como sustituto de eventos
```tsx
// ❌ MAL — useEffect para reaccionar a clicks/cambios del usuario
useEffect(() => {
  if (formularioEnviado) {  // ← Estado artificial para disparar un efecto
    guardarDatos();
  }
}, [formularioEnviado]);
```
**Regla**: Si es consecuencia de una acción del usuario, va en el event handler, NO en useEffect.

### ❌ Antipatrón 3: Context API como estado global mutable de alta frecuencia
```tsx
// ❌ MAL — Context para datos que cambian con cada keystroke
const FormContext = createContext({ nombreUsuario: "", onChange: () => {} });
// Cualquier cambio en un input re-renderiza TODOS los consumers
```

### ❌ Antipatrón 4: Props drilling excesivo (más de 2 niveles)
```tsx
// ❌ MAL — Pasar props por 3+ niveles de componentes intermediarios
<Parent usuario={usuario}>
  <Child usuario={usuario}>
    <GrandChild usuario={usuario} />  {/* ← GrandChild es el único que lo usa */}
  </Child>
</Parent>
```

---

## 3. Reglas de Oro

1. **Un componente = una responsabilidad**: Si un componente hace más de una cosa conceptual, dividirlo
2. **Custom Hooks para lógica reutilizable**: Nunca copiar lógica de estado entre componentes
3. **Memoización solo cuando se puede medir**: Usar `useMemo`/`useCallback`/`memo` con profiler, no por defecto
4. **Context API solo para estado de baja frecuencia**: Tema, idioma, usuario autenticado
5. **TypeScript strict**: Sin `any`, sin `as` innecesarios, interfaces explícitas para todas las props
6. **Derivar estado en lugar de sincronizarlo**: Si un valor se puede calcular desde otro estado, calcularlo

---

## 4. Custom Hook — Patrón Base

```typescript
// hooks/useAsync.ts — Hook genérico para operaciones asíncronas
import { useState, useCallback } from "react";

interface AsyncState<T> {
  data: T | null;
  loading: boolean;
  error: Error | null;
}

/**
 * Hook genérico para manejar operaciones asíncronas con estado de carga y error.
 * 
 * @param asyncFn - Función asíncrona a ejecutar
 * @returns Estado de la operación y función para ejecutarla
 */
export function useAsync<T, Args extends unknown[]>(
  asyncFn: (...args: Args) => Promise<T>
) {
  const [state, setState] = useState<AsyncState<T>>({
    data: null,
    loading: false,
    error: null,
  });

  const execute = useCallback(
    async (...args: Args) => {
      setState({ data: null, loading: true, error: null });
      try {
        const data = await asyncFn(...args);
        setState({ data, loading: false, error: null });
        return data;
      } catch (error) {
        const err = error instanceof Error ? error : new Error(String(error));
        setState({ data: null, loading: false, error: err });
        throw err;
      }
    },
    [asyncFn]
  );

  return { ...state, execute };
}

// Uso:
// const { data, loading, error, execute: cargarUsuarios } = useAsync(usuariosApi.listar);
// await cargarUsuarios({ pagina: 1 });
```

---

## 5. Context API — Patrón con Reducer

```typescript
// features/auth/context/AuthContext.tsx
import { createContext, useContext, useReducer, ReactNode } from "react";

// Tipos explícitos
interface Usuario {
  id: string;
  email: string;
  rol: "admin" | "operador" | "visor";
}

interface AuthState {
  usuario: Usuario | null;
  cargando: boolean;
}

type AuthAction =
  | { type: "LOGIN_EXITOSO"; payload: Usuario }
  | { type: "LOGOUT" }
  | { type: "CARGANDO"; payload: boolean };

// Reducer puro y predecible
function authReducer(state: AuthState, action: AuthAction): AuthState {
  switch (action.type) {
    case "LOGIN_EXITOSO":
      return { ...state, usuario: action.payload, cargando: false };
    case "LOGOUT":
      return { usuario: null, cargando: false };
    case "CARGANDO":
      return { ...state, cargando: action.payload };
    default:
      return state;
  }
}

const AuthContext = createContext<{
  state: AuthState;
  dispatch: React.Dispatch<AuthAction>;
} | null>(null);

export function AuthProvider({ children }: { children: ReactNode }) {
  const [state, dispatch] = useReducer(authReducer, { usuario: null, cargando: true });
  return (
    <AuthContext.Provider value={{ state, dispatch }}>
      {children}
    </AuthContext.Provider>
  );
}

// Hook tipado — nunca usar useContext directamente fuera de este módulo
export function useAuth() {
  const context = useContext(AuthContext);
  if (!context) throw new Error("useAuth debe usarse dentro de <AuthProvider>");
  return context;
}
```

---

## 6. Servicio API — Patrón Cliente HTTP

```typescript
// services/api/client.ts
// CONFIG: Ajustar BASE_URL según entorno
const BASE_URL = import.meta.env.VITE_API_BASE_URL ?? "http://localhost:8000/api/v1";

interface ApiOptions extends RequestInit {
  token?: string;
}

async function apiRequest<T>(
  endpoint: string,
  options: ApiOptions = {}
): Promise<T> {
  const { token, ...fetchOptions } = options;
  const headers: HeadersInit = {
    "Content-Type": "application/json",
    ...(token ? { Authorization: `Bearer ${token}` } : {}),
    ...options.headers,
  };

  const response = await fetch(`${BASE_URL}${endpoint}`, {
    ...fetchOptions,
    headers,
  });

  if (!response.ok) {
    const error = await response.json().catch(() => ({}));
    throw new Error(error.detail ?? `HTTP ${response.status}: ${response.statusText}`);
  }

  if (response.status === 204) return undefined as unknown as T;
  return response.json() as Promise<T>;
}

export const api = {
  get: <T>(url: string, token?: string) =>
    apiRequest<T>(url, { method: "GET", token }),
  post: <T>(url: string, body: unknown, token?: string) =>
    apiRequest<T>(url, { method: "POST", body: JSON.stringify(body), token }),
  put: <T>(url: string, body: unknown, token?: string) =>
    apiRequest<T>(url, { method: "PUT", body: JSON.stringify(body), token }),
  delete: <T>(url: string, token?: string) =>
    apiRequest<T>(url, { method: "DELETE", token }),
};
```

---

## 7. Checklist de Verificación

### Arquitectura
- [ ] Separación clara entre `components/` (UI pura) y `features/` (dominio + estado)
- [ ] Sin lógica de negocio en componentes — delegada a custom hooks o services
- [ ] Custom hooks para cualquier lógica que se repita en 2+ componentes

### TypeScript
- [ ] Sin `any` en el código fuente (configurar `"noImplicitAny": true`)
- [ ] Todas las props con interfaces TypeScript explícitas
- [ ] Respuestas de API tipadas con interfaces

### Performance
- [ ] `useMemo`/`useCallback` solo donde el profiler muestra re-renders costosos
- [ ] Listas largas con virtualización (react-window) si superan ~200 items
- [ ] Imágenes con lazy loading (`loading="lazy"`)

### Accesibilidad
- [ ] Todos los elementos interactivos tienen `aria-label` descriptivo
- [ ] Formularios con `<label htmlFor>` asociado a cada input
- [ ] Contraste de colores ≥ 4.5:1 (WCAG AA)
