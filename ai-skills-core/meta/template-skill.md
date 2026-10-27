---
# =============================================================================
# SKILL-SPEC-v1 — Plantilla Maestra de Skills de IA
# Copiar este archivo, renombrarlo y rellenarlo para crear una nueva skill.
# Convención de nombre: <dominio>-<tecnologia>-<patron>.md
# Ejemplo: django-drf-performance-optimization.md
# =============================================================================

id: "<dominio>-<tecnologia>-<patron>"          # Kebab-case canónico único
version: "1.0.0"                                # SemVer: MAJOR.MINOR.PATCH
domain: "Backend | Frontend | DevOps | Data | Automation"  # Elegir uno
technologies:
  - "tecnologia-1"
  - "tecnologia-2"
triggers:
  - "Cuándo la IA o el desarrollador debe activar esta skill (condición 1)"
  - "Cuándo la IA o el desarrollador debe activar esta skill (condición 2)"
  - "Síntoma o patrón de código que indica que esta skill es relevante"
related_skills:
  - "id-de-skill-relacionada-1"
  - "id-de-skill-relacionada-2"
antipatterns_addressed:
  - "Nombre del antipatrón que esta skill previene"
supersedes: ""                                  # ID de skill que esta reemplaza (si aplica)
source_generation: "core"                       # g1 | g2 | core
---

# [Título Descriptivo de la Skill]

> **TL;DR para IA:** [Una sola oración que resuma qué hace esta skill y cuándo usarla.]

---

## 1. Propósito y Contexto Arquitectónico

### Problema que Resuelve
[Describir el dolor de ingeniería que motivó esta skill. Ser específico sobre el contexto
en que aparece el problema y las consecuencias de no abordarlo correctamente.]

### Cuándo Aplicar (Triggers Detallados)
- ✅ **Activar cuando**: [condición concreta y observable]
- ✅ **Activar cuando**: [condición concreta y observable]
- ❌ **No activar si**: [contraejemplo o caso excluido]

### Relación con Otras Skills
- **Prerequisito de**: `[skill-id]` — [razón]
- **Complementa**: `[skill-id]` — [razón]
- **Alternativa a**: `[skill-id]` cuando [condición]

---

## 2. Antipatrones a Evitar (Errores de Batalla)

> ⚠️ Esta sección es crítica. Son lecciones aprendidas de producción.

### ❌ Antipatrón 1: [Nombre descriptivo]
```[lenguaje]
# ❌ MAL — Descripción del problema
codigo_malo_aqui()
```
**Por qué falla**: [Explicación técnica del fallo]
**Consecuencia**: [Qué pasa en producción si se ignora]

### ❌ Antipatrón 2: [Nombre descriptivo]
```[lenguaje]
# ❌ MAL — Descripción del problema
otro_codigo_malo()
```
**Por qué falla**: [Explicación técnica del fallo]

---

## 3. Reglas de Oro (Golden Rules)

> Directrices estrictas que NUNCA deben violarse al aplicar esta skill.

1. **[Regla 1]**: [Descripción imperativa concisa]
2. **[Regla 2]**: [Descripción imperativa concisa]
3. **[Regla 3]**: [Descripción imperativa concisa]
4. **[Regla 4]**: [Descripción imperativa concisa]
5. **[Regla 5]**: [Descripción imperativa concisa]

---

## 4. Guía de Implementación Paso a Paso

### Paso 1: [Nombre del paso]
[Instrucción accionable y concreta]

```[lenguaje]
# ✅ BIEN — Descripción del patrón correcto
codigo_correcto_aqui()
```

### Paso 2: [Nombre del paso]
[Instrucción accionable y concreta]

```[lenguaje]
# ✅ BIEN
mas_codigo_correcto()
```

### Paso 3: [Nombre del paso]
[Instrucción accionable y concreta]

---

## 5. Patrón de Implementación (Boilerplate Completo)

> Código limpio, tipado, modular y comentado. Sin nombres de clientes ni hardcoding.
> Variables de configuración marcadas con `# CONFIG:` para fácil identificación.

```[lenguaje]
# ===========================================================================
# [Nombre del Módulo] — Boilerplate SKILL-SPEC-v1
# Skill: [skill-id]
# Parametrizable: sustituir valores marcados con CONFIG:
# ===========================================================================

CONFIG_PARAM_1 = "valor_por_defecto"   # CONFIG: ajustar según proyecto
CONFIG_PARAM_2 = 100                   # CONFIG: ajustar según proyecto

def funcion_principal(param1: str, param2: int) -> dict:
    """
    Descripción clara de qué hace esta función.
    
    Args:
        param1: Descripción del parámetro
        param2: Descripción del parámetro
        
    Returns:
        dict: Descripción del valor de retorno
        
    Raises:
        ValueError: Cuándo se lanza esta excepción
    """
    # Implementación limpia y comentada
    resultado = {}
    return resultado
```

---

## 6. Variables de Configuración y Parametrización

| Variable | Tipo | Valor por Defecto | Descripción |
|---|---|---|---|
| `CONFIG_PARAM_1` | `str` | `"valor"` | Qué configura este parámetro |
| `CONFIG_PARAM_2` | `int` | `100` | Qué configura este parámetro |

---

## 7. Checklist de Verificación

Antes de marcar esta skill como aplicada correctamente, verificar cada punto:

### Funcionalidad
- [ ] El código ejecuta sin errores en el entorno de desarrollo
- [ ] Los casos de uso principales están cubiertos
- [ ] Los casos edge están manejados explícitamente

### Calidad de Código
- [ ] Sin hardcoding de valores de configuración, credenciales o nombres de cliente
- [ ] Tipado explícito en todas las funciones y clases (type hints / TypeScript types)
- [ ] Docstrings/JSDoc en funciones públicas con Args, Returns y Raises
- [ ] Nombres de variables y funciones descriptivos en inglés

### Seguridad
- [ ] Sin credenciales, tokens o secretos en el código fuente
- [ ] Validación de entrada en todos los puntos de entrada externos
- [ ] Errores manejados sin exponer información sensible en mensajes públicos

### Testing
- [ ] Existe al menos un test unitario por función crítica
- [ ] Los antipatrones documentados están cubiertos como casos de test negativos
- [ ] El código puede ejecutarse en entorno de desarrollo sin servicios externos (mocks)

### Documentación
- [ ] El frontmatter YAML está completo y actualizado
- [ ] Los triggers son suficientemente descriptivos para activación autónoma por IA
- [ ] Los antipatrones incluyen código de ejemplo de lo que NO hacer

---

## 8. Referencias y Recursos

- [Documentación Oficial](https://url-de-documentacion.com)
- [Skill Relacionada](../ruta/a/otra-skill/SKILL.md)
- [RFC / ADR de Referencia](#)

---

*Skill creada: YYYY-MM-DD · Última revisión: YYYY-MM-DD · Autor: [equipo]*
