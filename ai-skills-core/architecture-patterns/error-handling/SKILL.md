---
id: "error-handling-patterns"
version: "1.0.0"
domain: "Architecture"
technologies:
  - "Python 3.11+"
  - "TypeScript"
  - "Structured JSON Logging"
  - "Tenacity (Retry / Backoff)"
  - "HTTP RFC 7807 (Problem Details)"
triggers:
  - "Diseñar jerarquía de excepciones en un proyecto"
  - "Implementar logging estructurado para observabilidad (Cloud Run, Datadog, GCP)"
  - "Consumir APIs externas propensas a caídas o rate-limits"
  - "Unificar formato de respuestas de error en endpoints REST"
related_skills:
  - "clean-architecture-patterns"
  - "security-hardening"
  - "django-drf-patterns"
source_generation: "core"
---

# Manejo de Errores, Resiliencia y Logging Estructurado

> **TL;DR para IA:** Aplica esta skill para diseñar excepciones de dominio jerárquicas, estandarizar logs en JSON para trazabilidad en la nube y blindar integraciones mediante reintentos con backoff exponencial.

---

## 1. Jerarquía de Excepciones de Dominio

Nunca captures ni relances excepciones genéricas (`except Exception:` sin re-clasificar). Define una jerarquía limpia donde cada error expone su contexto y su código de error técnico.

```
AppBaseException
├── DomainException (Errores de regla de negocio -> HTTP 4xx)
│   ├── ResourceNotFoundException (404)
│   ├── InvalidBusinessOperationException (422)
│   └── UnauthorizedAccessException (403)
└── InfrastructureException (Fallos técnicos de red, BD, IO -> HTTP 5xx)
    ├── ExternalServiceTimeoutException (504)
    ├── DatabaseConnectionException (503)
    └── ConcurrencyLockException (409)
```

---

## 2. Antipatrones a Evitar

### ❌ Antipatrón 1: Captura ciega y silenciamiento de errores
```python
# ❌ MAL: Silencia el error, oculta la causa raíz y dificulta el triage
try:
    procesar_pago()
except Exception:
    pass  # o print("error")
```

### ❌ Antipatrón 2: Logs en texto plano desestructurados
```python
# ❌ MAL: Imposible filtrar por cliente, correlación o severidad en GCP Logs / Datadog
logging.info(f"Error procesando orden para el usuario {user_id}")
```

### ❌ Antipatrón 3: Reintentos infinitos inmediatos (Thundering Herd)
```python
# ❌ MAL: Reintentar de inmediato satura aún más el servicio caído
while True:
    try:
        return api_call()
    except Exception:
        continue
```

---

## 3. Reglas de Oro

1. **Excepciones de Dominio Tipadas**: Las capas internas lanzan excepciones propias del dominio, nunca errores de driver (`psycopg2.OperationalError` debe mapearse a `DatabaseConnectionException`).
2. **Logs Estructurados en JSON**: Siempre emitir logs en JSON con `timestamp`, `severity`, `trace_id` y `context`.
3. **Reintentos con Backoff Exponencial y Jitter**: En llamadas de red a APIs externas, usar esperas crecientes con aleatoriedad para no colapsar el destino.
4. **Respuestas de Error bajo RFC 7807**: Endpoints REST deben responder con formato Problem Details estándar (`type`, `title`, `status`, `detail`, `instance`).

---

## 4. Patrón de Implementación: Jerarquía de Excepciones Tipadas

```python
# src/core/exceptions.py
from typing import Any

class AppBaseException(Exception):
    """Clase base de excepciones del sistema con metadatos contextuales."""
    def __init__(
        self,
        mensaje: str,
        codigo_error: str,
        status_http: int = 500,
        contexto: dict[str, Any] | None = None
    ):
        super().__init__(mensaje)
        self.mensaje = mensaje
        self.codigo_error = codigo_error
        self.status_http = status_http
        self.contexto = contexto or {}

class ResourceNotFoundException(AppBaseException):
    def __init__(self, entidad: str, identificador: str):
        super().__init__(
            mensaje=f"No se encontró el recurso '{entidad}' con ID '{identificador}'.",
            codigo_error="RESOURCE_NOT_FOUND",
            status_http=404,
            contexto={"entidad": entidad, "id": identificador}
        )

class InvalidBusinessRuleException(AppBaseException):
    def __init__(self, regla: str, detalle: str):
        super().__init__(
            mensaje=f"Violación de regla de negocio '{regla}': {detalle}",
            codigo_error="BUSINESS_RULE_VIOLATION",
            status_http=422,
            contexto={"regla": regla, "detalle": detalle}
        )
```

---

## 5. Resiliencia: Reintentos con Backoff Exponencial y Jitter

```python
# src/core/resilience.py
import random
import time
from functools import wraps
from typing import Callable, TypeVar, Any
import logging

T = TypeVar("T")
logger = logging.getLogger("resilience")

def con_reintentos(
    max_intentos: int = 3,
    base_delay: float = 1.0,
    max_delay: float = 10.0,
    excepciones_reintentables: tuple[type[Exception], ...] = (Exception,)
) -> Callable[[Callable[..., T]], Callable[..., T]]:
    """
    Decorador para reintentar operaciones transitorias con backoff exponencial y jitter.
    """
    def decorador(func: Callable[..., T]) -> Callable[..., T]:
        @wraps(func)
        def wrapper(*args: Any, **kwargs: Any) -> T:
            intento = 1
            while True:
                try:
                    return func(*args, **kwargs)
                except excepciones_reintentables as exc:
                    if intento >= max_intentos:
                        logger.error(
                            "Operacion agotada tras %d reintentos. Fallo final: %s",
                            max_intentos, exc
                        )
                        raise
                    
                    # Backoff exponencial con Full Jitter
                    delay_teorico = min(max_delay, base_delay * (2 ** (intento - 1)))
                    delay_real = random.uniform(0, delay_teorico)
                    
                    logger.warning(
                        "Fallo intento %d/%d para %s (%s). Esperando %.2fs...",
                        intento, max_intentos, func.__name__, exc, delay_real
                    )
                    time.sleep(delay_real)
                    intento += 1
        return wrapper
    return decorador
```

---

## 6. Checklist de Verificación

- [ ] Todas las excepciones expuestas en la API siguen la convención RFC 7807 (`type`, `title`, `status`, `detail`)
- [ ] No existen bloques `except:` vacíos o que simplemente usen `pass` sin logging explícito
- [ ] Los logs en entornos de producción emiten exclusivamente JSON estructurado
- [ ] Ningún log contiene información sensible (PII, contraseñas, tokens de autenticación o tarjetas de crédito)
- [ ] Los reintentos de red implementan límites estrictos (`max_intentos <= 4`) y backoff exponencial con jitter
