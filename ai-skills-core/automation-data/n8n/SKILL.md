---
id: "n8n-workflow-design"
version: "1.0.0"
domain: "Automation"
technologies:
  - "n8n"
  - "Webhooks HTTP"
  - "JavaScript (n8n expressions)"
  - "REST APIs"
triggers:
  - "Diseñar un flujo de automatización en n8n"
  - "Crear un webhook receptor en n8n"
  - "Implementar manejo de errores en un workflow n8n"
  - "Conectar n8n con una API externa (REST, Google, etc.)"
  - "Sub-workflows o workflows modulares en n8n"
  - "Retry logic en n8n ante fallas de API"
related_skills:
  - "error-handling-patterns"
  - "security-hardening"
  - "etl-pipeline-patterns"
source_generation: "core"
---

# n8n — Diseño de Workflows, Webhooks y Manejo de Errores

> **TL;DR para IA:** Carga esta skill al diseñar cualquier flujo n8n. Contiene los patrones
> de estructura, manejo de errores, webhooks seguros y sub-workflows que previenen fallos en producción.

---

## 1. Principios de Diseño de Workflows

### El Flujo Canónico

```
Trigger (Webhook/Schedule/Event)
  └── Validate Input              ← Siempre validar primero
       └── Transform Data         ← Normalizar antes de procesar
            └── Business Logic    ← Nodo(s) de lógica principal
                 └── Error Branch ← Manejo explícito de errores
                      └── Output  ← Respuesta / notificación
```

### Regla de Modularidad
- Un workflow = una responsabilidad de negocio
- Máximo ~15-20 nodos por workflow; extraer sub-workflows si crece más
- Nombres descriptivos en CADA nodo (no dejar "HTTP Request1", "Set2", etc.)

---

## 2. Antipatrones a Evitar

### ❌ Antipatrón 1: Workflow sin Error Handling
```
Webhook → HTTP Request → Google Sheets
# Si "HTTP Request" falla, el workflow termina silenciosamente
# No hay notificación, no hay registro del error
```
**Consecuencia**: Datos perdidos sin trazabilidad. Imposible saber cuántos mensajes fallaron.

### ❌ Antipatrón 2: Credenciales hardcodeadas en nodos HTTP
```json
// ❌ MAL — Token hardcodeado en el nodo
{
  "url": "https://api.ejemplo.com/datos",
  "headers": {
    "Authorization": "Bearer sk-mi-token-real-123"
  }
}
```
**Usar siempre**: Credentials de n8n o Variables de Entorno (`{{ $env.MI_TOKEN }}`).

### ❌ Antipatrón 3: Procesar listas con nodos secuenciales sin Split In Batches
```
Set (array de 500 items) → HTTP Request (llamada por item sin batching)
# 500 llamadas HTTP secuenciales = timeout o rate limit
```

### ❌ Antipatrón 4: Webhook sin validación de autenticidad
```
Webhook (sin secreto) → Procesar datos
# Cualquiera puede enviar datos al endpoint y disparar el flujo
```

---

## 3. Reglas de Oro

1. **Error Workflow global**: Configurar un "Error Workflow" en n8n que notifique por email/Slack al fallar cualquier workflow
2. **Credentials de n8n para secretos**: Nunca hardcodear tokens; usar el gestor de credentials
3. **Validar payload en el primer nodo**: Verificar que el input tiene los campos esperados antes de procesar
4. **Split In Batches para listas**: Procesar listas grandes en lotes de 10-50 para evitar timeouts
5. **Nombres descriptivos en nodos**: Cada nodo describe QUÉ hace: "Obtener facturas pendientes", no "HTTP Request1"
6. **Webhook con header secreto**: Verificar un header `X-Webhook-Secret` para autenticar el origen

---

## 4. Webhook Seguro — Patrón de Validación

```javascript
// Nodo: "Validar Webhook" (tipo: Code)
// Colocar inmediatamente después del nodo Webhook

const secret = $env.WEBHOOK_SECRET;                    // CONFIG: Variable de entorno
const receivedSecret = $input.first().json.headers['x-webhook-secret'];

if (!receivedSecret || receivedSecret !== secret) {
  throw new Error('Webhook: Autenticación fallida - secret inválido');
}

// Validar campos obligatorios del payload
const body = $input.first().json.body;
const camposObligatorios = ['evento', 'datos', 'timestamp'];  // CONFIG: ajustar campos

const faltantes = camposObligatorios.filter(campo => !(campo in body));
if (faltantes.length > 0) {
  throw new Error(`Payload incompleto. Campos faltantes: ${faltantes.join(', ')}`);
}

return $input.all();
```

---

## 5. Error Handling — Patrón Completo

### Estructura del Error Workflow

```
Error Trigger
  └── Code (formatear error)
       └── Switch (por tipo de error)
            ├── → Notify Slack (errores críticos)
            └── → Log to DB/Sheet (todos los errores)
```

```javascript
// Nodo: "Formatear Error" (tipo: Code) en el Error Workflow
const errorData = $input.first().json;

const mensajeError = {
  workflow_nombre: errorData.workflow.name,
  nodo_fallido: errorData.execution.lastNodeExecuted,
  mensaje: errorData.error.message,
  timestamp: new Date().toISOString(),
  execution_id: errorData.execution.id,
  // Incluir contexto relevante para debugging
  datos_entrada: JSON.stringify(errorData.error.context ?? {}).slice(0, 500), // Limitar tamaño
};

return [{ json: mensajeError }];
```

---

## 6. Sub-Workflows — Patrón de Modularidad

Cuando un flujo supera ~15 nodos o contiene lógica reutilizable, extraerla a un sub-workflow:

```javascript
// Nodo: "Llamar Sub-Workflow" (tipo: Execute Workflow)
// Configurar: workflowId = ID del sub-workflow, mode = "each item"

// El sub-workflow debe:
// 1. Recibir un trigger tipo "Execute Workflow Trigger"  
// 2. Retornar datos con un nodo "Respond to Webhook" o simplemente terminar
// 3. Documentar su contrato (inputs/outputs esperados) en su descripción
```

**Sub-workflows recomendados para extraer:**
- `sw-enviar-notificacion`: Lógica de emails/Slack reutilizable
- `sw-validar-datos`: Validación común a múltiples flujos
- `sw-log-actividad`: Registro de auditoría centralizado

---

## 7. Procesamiento de Listas con Batching

```
HTTP Request (obtener lista grande)
  └── Split In Batches (tamaño: 25)        ← CONFIG: 10-50 según rate limits de la API
       └── HTTP Request (procesar item)
            └── Wait (1 segundo)           ← CONFIG: ajustar según rate limit de la API destino
                 └── Merge (cuando terminen todos los batches)
                      └── Siguiente nodo
```

---

## 8. Checklist de Verificación

### Seguridad
- [ ] Todos los tokens y secretos en Credentials de n8n o variables de entorno
- [ ] Webhooks públicos con validación de header secreto
- [ ] Sin datos sensibles (emails, IDs privados) en nombres de nodos o notas visibles en logs

### Confiabilidad
- [ ] Error Workflow configurado a nivel de instancia n8n
- [ ] Todos los nodos HTTP tienen timeout configurado (máx 30s recomendado)
- [ ] Listas de más de 50 items procesadas con "Split In Batches"
- [ ] Retry configurado en nodos HTTP (máx 3 intentos, backoff exponencial)

### Mantenibilidad
- [ ] Cada nodo tiene un nombre descriptivo en español/inglés consistente
- [ ] Workflows complejos tienen notas (sticky notes) explicando la lógica
- [ ] Sub-workflows documentan sus inputs y outputs esperados
- [ ] Variables de configuración centralizadas al inicio del workflow (nodo Set)
