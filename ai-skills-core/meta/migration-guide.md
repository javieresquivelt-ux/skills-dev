# Guía de Migración — De Skills_g1/g2 a ai-skills-core

## Objetivo

Transferir el conocimiento de las generaciones anteriores a la nueva estructura
sin perder aprendizajes, y normalizando el formato al estándar SKILL-SPEC-v1.

---

## Mapa de Migración Completo

| Skill Original | Generación | Destino en ai-skills-core | Estado |
|---|---|---|---|
| `gas-rest-api-design` | g1 | `dev-stack/frontend-core/SKILL.md` (Sección 4) | ✅ Consolidado |
| `sheets-as-database` | g1 | `dev-stack/frontend-core/SKILL.md` (Sección 4) | ✅ Consolidado |
| `timezone-wallclock-engine` | g1 | `dev-stack/frontend-core/SKILL.md` (Sección 5) | ✅ Consolidado |
| `serverless-stateless-auth` | g1 | `architecture-patterns/security/SKILL.md` (Sección 6) | ✅ Consolidado |
| `config-driven-dynamic-ui` | g1 | `dev-stack/frontend-core/SKILL.md` (Sección 1) | ✅ Consolidado |
| `gas-node-testing-harness` | g1 | `dev-stack/frontend-core/SKILL.md` (Sección 4) | ✅ Consolidado |
| `dual-repo-static-spa-deployment` | g1 | `devops-cloud/docker/SKILL.md` (Sección 5) | ✅ Consolidado |
| `sass-7-1-architecture` | g2 | `dev-stack/react/sass-7-1-architecture/SKILL.md` | ✅ Migrado |
| `modular-rbac-spa` | g2 | `dev-stack/frontend-core/SKILL.md` (Sección 2) | ✅ Consolidado |
| `local-first-hybrid-storage` | g2 | `dev-stack/frontend-core/SKILL.md` (Sección 3) | ✅ Consolidado |
| `google-sheets-baas` | g2 | `dev-stack/frontend-core/SKILL.md` (Sección 4) | ✅ Consolidado |
| `google-identity-gis-auth` | g2 | `architecture-patterns/security/SKILL.md` (Sección 5) | ✅ Consolidado |
| `business-alert-engine` | g2 | `architecture-patterns/clean-architecture/SKILL.md` (Sección 5) | ✅ Consolidado |
| `dual-repo-github-pages-deploy` | g2 | `devops-cloud/docker/SKILL.md` (Sección 5) | ✅ Consolidado |
| `ai-pair-programming-harness` | g2 | `meta/CONTRIBUTING.md` (Sección HARNESS) | ✅ Migrado |

---

## Protocolo de Migración por Skill

Para cada skill marcada como ⬜ Pendiente, seguir estos pasos:

### Paso 1: Auditoría de Contenido Original
```bash
# Leer el SKILL.md original completo
cat Skills_g1/<nombre-skill>/SKILL.md
# o
cat Skills_g2/<nombre-skill>/SKILL.md
```

**Extraer y anotar**:
- [ ] Todas las **Reglas de Oro** (Golden Rules)
- [ ] Todos los **Antipatrones** con sus ejemplos de código
- [ ] Todos los **Snippets de código** reutilizables
- [ ] Los **Triggers** específicos
- [ ] El **Checklist de verificación**

### Paso 2: Identificar Solapamientos

Comparar con skills ya existentes en ai-skills-core:

```bash
# Buscar términos clave en el repositorio nuevo
grep -r "término-clave" ai-skills-core/
```

Si existe solapamiento:
- **Fusionar**: Combinar ambas versiones eligiendo el contenido más maduro
- **Referenciar**: Si son complementarias, añadir una sección `related_skills`
- **Deprecar**: Si una es superconjunto de la otra, marcar la menor como deprecated

### Paso 3: Normalizar al Formato SKILL-SPEC-v1

Checklist de normalización:
- [ ] Agregar frontmatter YAML completo con `id`, `version`, `domain`, `technologies`, `triggers`
- [ ] Eliminar referencias a nombres de clientes o proyectos específicos
- [ ] Parametrizar todos los valores hardcodeados (marcar con `# CONFIG:`)
- [ ] Convertir código de cliente-específico a código genérico con parámetros
- [ ] Verificar que el código de boilerplate es ejecutable fuera de contexto de un proyecto
- [ ] Añadir `source_generation: "g1" | "g2"` en el frontmatter

### Paso 4: Verificar Deduplicación entre g1 y g2

Comparar skills que tratan el mismo tema:

| Par a Revisar | Tema en Común | Decisión Recomendada |
|---|---|---|
| `sheets-as-database` (g1) vs `google-sheets-baas` (g2) | Google Sheets como BD | Fusionar en una sola skill madura |
| `dual-repo-static-spa-deployment` (g1) vs `dual-repo-github-pages-deploy` (g2) | Dual-repo deployment | Fusionar, g2 es más maduro |
| `serverless-stateless-auth` (g1) vs `google-identity-gis-auth` (g2) | Auth serverless | Mantener separadas: g1 = HMAC general, g2 = GIS específico |

### Paso 5: Actualizar el README Maestro

Después de migrar, actualizar el estado en este archivo a ✅ Migrado y añadir la
skill a la tabla correspondiente en `README.md`.

---

## Script de Migración Asistida

```bash
#!/usr/bin/env bash
# migrate-skill.sh — Asistente de migración de skills
# Uso: ./migrate-skill.sh <ruta-origen> <ruta-destino>

SOURCE=$1
DEST=$2

if [ -z "$SOURCE" ] || [ -z "$DEST" ]; then
  echo "Uso: ./migrate-skill.sh <origen> <destino>"
  exit 1
fi

# Crear directorio destino si no existe
mkdir -p "$(dirname "$DEST")"

# Copiar skill original como punto de partida
cp "$SOURCE" "$DEST"

echo "✅ Skill copiada a $DEST"
echo "📋 Próximos pasos manuales:"
echo "  1. Actualizar frontmatter YAML al formato SKILL-SPEC-v1"
echo "  2. Eliminar referencias a clientes o proyectos específicos"
echo "  3. Parametrizar valores hardcodeados (marcar con # CONFIG:)"
echo "  4. Verificar boilerplate ejecutable"
echo "  5. Completar checklist de verificación"
echo "  6. Actualizar migration-guide.md → cambiar ⬜ a ✅"
```

---

## Criterios de Calidad Post-Migración

Una skill migrada se considera **correctamente procesada** cuando:

1. ✅ No contiene ningún nombre de empresa, cliente o dominio específico real
2. ✅ El boilerplate se puede copiar a un proyecto nuevo sin modificaciones estructurales
3. ✅ Los triggers son lo suficientemente descriptivos para activación autónoma por un LLM
4. ✅ Los antipatrones incluyen ejemplos de código negativo Y la corrección positiva
5. ✅ El frontmatter YAML es válido y completo
6. ✅ Está referenciada en el `README.md` maestro en la tabla correcta

