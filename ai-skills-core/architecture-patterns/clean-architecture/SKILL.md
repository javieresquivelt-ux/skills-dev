---
id: "clean-architecture-patterns"
version: "1.0.0"
domain: "Architecture"
technologies:
  - "Python / TypeScript"
  - "SOLID Principles"
  - "Repository Pattern"
  - "Service Layer"
  - "Domain-Driven Design (Lite)"
triggers:
  - "Diseñar o refactorizar la arquitectura interna de un módulo de software"
  - "Desacoplar la lógica de negocio de frameworks web (Django, Express, FastAPI, React)"
  - "Implementar un motor de reglas o alertas puro en memoria"
  - "Invertir dependencias de persistencia o APIs externas mediante interfaces/protocolos"
related_skills:
  - "error-handling-patterns"
  - "django-drf-patterns"
  - "react-component-patterns"
source_generation: "core"
---

# Clean Architecture — Capas, Inversión de Dependencias y Motores Puros

> **TL;DR para IA:** Usa esta skill para separar la lógica de negocio del framework y los detalles de infraestructura. El dominio no debe depender de bases de datos, APIs externas ni interfaces de usuario.

---

## 1. Modelo de Capas y Regla de Dependencia

La regla fundamental de Clean Architecture es que **las dependencias en el código fuente solo pueden apuntar hacia adentro**, hacia las políticas de mayor nivel de abstracción:

```
┌────────────────────────────────────────────────────────┐
│ 1. Frameworks & Drivers (Web, DB, UI, Dispositivos)     │
│    ┌────────────────────────────────────────────────┐  │
│    │ 2. Interface Adapters (Controllers, Repos, DTO)│  │
│    │    ┌──────────────────────────────────────┐    │  │
│    │    │ 3. Application Business Rules (Use   │    │  │
│    │    │    Cases / Services)                 │    │  │
│    │    │    ┌────────────────────────────┐    │    │  │
│    │    │    │ 4. Enterprise Domain Rules │    │    │  │
│    │    │    │    (Entities, Pure Engines)│    │    │  │
│    │    │    └────────────────────────────┘    │    │  │
│    │    └──────────────────────────────────────┘    │  │
│    └────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────┘
```

1. **Entities & Domain Engines**: Modelos de datos puros y reglas comerciales independientes de cualquier tecnología.
2. **Use Cases / Service Layer**: Orquestan el flujo de datos hacia y desde las entidades.
3. **Interface Adapters**: Convierten datos del formato de los casos de uso al formato de la web/BD.
4. **Frameworks & Infrastructure**: Django, React, Postgres, BigQuery, SDKs externos.

---

## 2. Antipatrones a Evitar

### ❌ Antipatrón 1: Entidades de dominio atadas al ORM
```python
# ❌ MAL: Lógica de negocio mezclada con queries SQL y detalles de base de datos
class Factura(models.Model):
    total = models.DecimalField(...)
    
    def emitir(self):
        # Efecto colateral directo: dependencia oculta de red / BD
        requests.post("https://api.sat.gob/certificar", json={"monto": str(self.total)})
        self.estado = "EMITIDA"
        self.save()
```
**Consecuencia**: Imposible testear la lógica de negocio sin conexión a red o base de datos levantada.

### ❌ Antipatrón 2: Lógica de negocio en controladores/vistas
```typescript
// ❌ MAL: El controlador web valida negocio, calcula márgenes y muta la BD
export async function crearPedidoHandler(req: Request, res: Response) {
  const { items, clienteId } = req.body;
  if (items.length > 50) return res.status(400).send("Límite excedido");
  const descuento = clienteId === "VIP" ? 0.2 : 0;
  // ... cálculos directos aquí
}
```

---

## 3. Reglas de Oro

1. **Motores de Dominio como Funciones Puras**: Entradas inmutables $\to$ cálculo determinista $\to$ salida inmutable. Sin I/O ni llamadas asíncronas dentro del core.
2. **Inversión de Dependencias (DIP)**: Los Casos de Uso dependen de abstracciones (Interfaces / Protocols / Abstract Classes), no de implementaciones concretas.
3. **Independencia del Framework**: Si cambias Django por FastAPI o React por Vue, el 100% del código en `domain/` y `services/` debe permanecer inalterado.
4. **DTOs en los límites**: Utilizar objetos de transferencia de datos limpios (Pydantic / dataclasses / interfaces TS) al cruzar fronteras de capas.

---

## 4. Patrón de Implementación: Inversión de Dependencias (Python)

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass
from typing import Protocol
import uuid

# --- CAPA 1: DOMINIO (Pure Domain Entity) ---
@dataclass(frozen=True)
class Orden:
    id: str
    monto_total: float
    cliente_email: str
    estado: str = "PENDIENTE"

# --- CAPA 2: PUERTO / CONTRATO (Dependency Inversion) ---
class OrdenRepository(Protocol):
    """Contrato que cualquier almacén de datos debe satisfacer."""
    def guardar(self, orden: Orden) -> None:
        ...
    def buscar_por_id(self, orden_id: str) -> Orden | None:
        ...

class Notificador(Protocol):
    def notificar_creacion(self, orden: Orden) -> None:
        ...

# --- CAPA 3: CASO DE USO / SERVICIO DE APLICACIÓN ---
class CrearOrdenUseCase:
    def __init__(self, repo: OrdenRepository, notificador: Notificador):
        self._repo = repo
        self._notificador = notificador

    def ejecutar(self, monto: float, email: str) -> Orden:
        if monto <= 0:
            raise ValueError("El monto de la orden debe ser positivo")
            
        orden = Orden(
            id=str(uuid.uuid4()),
            monto_total=monto,
            cliente_email=email,
            estado="CREADA"
        )
        self._repo.guardar(orden)
        self._notificador.notificar_creacion(orden)
        return orden
```

---

## 5. Patrón Especializado: Motor de Alertas de Negocio Puro

> Consolidado y abstraído de `Skills_g2/business-alert-engine`

Un motor de reglas y alertas jamás debe ejecutar consultas directas en su ciclo de evaluación. Recibe un snapshot de datos y genera alertas inmutables:

```typescript
// --- MODELOS INMUTABLES ---
export type SeveridadAlerta = "INFO" | "WARNING" | "CRITICAL";

export interface SnapshotComercial {
  clienteId: string;
  saldoPendiente: number;
  diasMora: number;
  tieneAcuerdoEspecial: boolean;
}

export interface Alerta {
  codigo: string;
  severidad: SeveridadAlerta;
  mensaje: string;
  autoExtinguible: boolean;
}

// --- MOTOR PURO EN MEMORIA ---
export function evaluarAlertasComerciales(snapshot: SnapshotComercial): Alerta[] {
  const alertas: Alerta[] = [];

  // Regla 1: Saldo en mora con excepción de acuerdos especiales
  if (snapshot.saldoPendiente > 0 && snapshot.diasMora > 30) {
    if (snapshot.tieneAcuerdoEspecial) {
      alertas.push({
        codigo: "MORA_CON_ACUERDO",
        severidad: "INFO",
        mensaje: `Cliente con mora de ${snapshot.diasMora} días bajo convenio especial acordado.`,
        autoExtinguible: true,
      });
    } else {
      alertas.push({
        codigo: "MORA_CRITICA",
        severidad: "CRITICAL",
        mensaje: `Mora de ${snapshot.diasMora} días supera el umbral operativo permitido.`,
        autoExtinguible: false,
      });
    }
  }

  // Regla 2: Auto-extinción cuando el saldo es conciliado ($0)
  if (snapshot.saldoPendiente <= 0) {
    return []; // No genera alertas; extingue automáticamente las previas
  }

  return Object.freeze(alertas);
}
```

---

## 6. Checklist de Verificación

- [ ] Los archivos de dominio (`domain/`) no importan Django, FastAPI, React, SQL ni librerías de red
- [ ] Los casos de uso interactúan con persistencia exclusivamente a través de interfaces (`Protocols` o `Abstract Classes`)
- [ ] Los tests unitarios de las entidades y servicios de aplicación corren en memoria en < 500ms sin levantar contenedores ni bases de datos
- [ ] Los motores de evaluación de reglas son funciones puras (sin `Date.now()` sin inyectar, sin `Math.random()`, sin I/O)
- [ ] Las dependencias son inyectadas en constructores (Dependency Injection) en lugar de instanciadas dentro de las clases
