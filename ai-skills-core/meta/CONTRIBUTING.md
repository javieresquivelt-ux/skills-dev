# Guía de Contribución — ai-skills-core

## Estándares de Calidad

### ✅ Criterios de Aceptación para una Nueva Skill

Una skill es aceptada en este repositorio si cumple **todos** los criterios:

1. **Formato SKILL-SPEC-v1**: Usa el `template-skill.md` sin omitir secciones
2. **Frontmatter completo**: `id`, `version`, `domain`, `technologies`, `triggers` y `related_skills` son obligatorios
3. **Sin hardcoding**: Sin nombres de clientes, dominios específicos, credenciales ni IPs
4. **Código ejecutable**: El boilerplate de la Sección 5 debe ser funcional y tipado
5. **Antipatrones con código**: Sección 2 incluye ejemplos de código negativo (❌) y positivo (✅)
6. **Checklist completo**: Todos los ítems de la Sección 7 están cubiertos por la implementación

---

## Proceso de Contribución

### Crear una Skill Nueva

```bash
# 1. Copiar la plantilla al directorio de dominio correspondiente
cp meta/template-skill.md dev-stack/python/mi-nueva-skill/SKILL.md

# 2. Editar el archivo con el contenido real
# 3. Verificar el checklist interno del SKILL.md antes de commitear
# 4. Actualizar el README.md del repositorio con la nueva entrada en la tabla
```

### Estructura de Directorios para una Skill con Recursos

```
dev-stack/django/mi-skill/
├── SKILL.md           ← Obligatorio: documentación canónica
├── examples/          ← Opcional: ejemplos funcionales
│   └── ejemplo.py
└── references/        ← Opcional: documentación adicional
    └── notas.md
```

---

## Marco HARNESS para Pair Programming con IA

> Heredado de `Skills_g2/ai-pair-programming-harness`

El framework HARNESS define el ciclo de vida de trabajo con agentes IA:

```
ANALIZAR → PLANIFICAR → APROBAR → EJECUTAR → REGISTRAR → APRENDER
```

### Reglas de Gobernanza

| Regla | Descripción |
|---|---|
| **Punto de Control Humano** | El agente NUNCA ejecuta cambios destructivos sin aprobación explícita |
| **Memoria Persistente** | Mantener `memory.md` y `task.md` actualizados al final de cada sesión |
| **Restricción de Regresión** | El agente propone, el humano aprueba cualquier cambio a código ya validado |
| **Contexto Explícito** | Al iniciar sesión, el agente recibe el estado actual desde `memory.md` |
| **Aprendizaje Acumulativo** | Los errores corregidos se documentan como antipatrones en la skill relevante |

### Archivos de Sesión (usar en cada proyecto)

```markdown
# memory.md — Estado actual del proyecto
## Contexto
## Decisiones Tomadas
## Pendientes

# task.md — Tarea actual
## Objetivo
## Pasos Completados
## Próximos Pasos
## Bloqueos
```

---

## Convenciones de Nomenclatura

| Elemento | Convención | Ejemplo |
|---|---|---|
| `id` de skill | `<dominio>-<tecnologia>-<patron>` en kebab-case | `django-drf-serializer-optimization` |
| Carpeta de skill | Mismo que `id` | `django-drf-serializer-optimization/` |
| Archivo principal | Siempre `SKILL.md` | `SKILL.md` |
| Variables en código | `UPPER_SNAKE_CASE` para constantes, `snake_case` para variables | `MAX_RETRIES`, `retry_count` |
| Comentarios de config | `# CONFIG:` para parámetros ajustables | `# CONFIG: ajustar según entorno` |

---

## Taxonomía de Dominios

| Dominio | Carpeta Base | Ejemplos de Skills |
|---|---|---|
| `Backend` | `dev-stack/django/`, `dev-stack/python/` | ORM, serializers, auth |
| `Frontend` | `dev-stack/react/`, `dev-stack/frontend-core/` | Hooks, RBAC, Sass |
| `DevOps` | `devops-cloud/` | Docker, GCP, DNS, CI/CD |
| `Data` | `automation-data/etl-pipelines/`, `automation-data/data-engineering/` | ETL, BigQuery, modelado |
| `Automation` | `automation-data/n8n/` | Workflows, webhooks |
| `Architecture` | `architecture-patterns/` | SOLID, security, errores |

