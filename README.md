# 🧠 AI Skills Core & Architecture Engine

> Repositorio centralizado de **Skills de IA, Patrones de Arquitectura de Software y Gobernanza Técnica** para el desarrollo asistido por Inteligencia Artificial y agentes autónomos.

Este repositorio actúa como la **Base de Conocimiento y Fuente Única de Verdad (Single Source of Truth)** para inicializar, guiar y estandarizar cualquier proyecto de software moderno (Fullstack, Backend, Frontend, Cloud/DevOps, Pipelines de Datos y Flujos de Automatización).

---

## 🏗️ Estructura del Repositorio

```text
.
├── README.md                      # Este documento (visión general del repositorio)
├── .gitignore                     # Filtros para seguridad, secretos y dependencias
│
├── ai-skills-core/                # Núcleo canónico de skills normalizadas (SKILL-SPEC-v1)
│   ├── README.md                  # Índice maestro y matriz de enrutamiento por proyecto
│   ├── meta/                      # Plantillas canónicas, guías de contribución y migración
│   │   ├── template-skill.md      # Plantilla oficial SKILL-SPEC-v1
│   │   ├── CONTRIBUTING.md        # Estándares y metodología HARNESS
│   │   └── migration-guide.md     # Bitácora histórica de consolidación
│   ├── dev-stack/                 # Python, Django DRF, React 18+, Sass 7-1, Frontend Core
│   ├── devops-cloud/              # GCP (Cloud Run, BigQuery), Docker, Easypanel, Hostinger DNS
│   ├── automation-data/           # n8n Workflows, Python ETLs, Data Engineering
│   └── architecture-patterns/     # Clean Architecture, Security Hardening, Error Handling
│
└── prompt/                        # Prompts maestros operacionales
    ├── prompt_skills_maestras.md  # Inyectar en agent.md / CLAUDE.md / .cursorrules de nuevos proyectos
    └── prompt_nuevos_skills.md    # Para ingestar y consolidar nuevos aprendizajes (ej. Skills_gX)
```

---

## ⚡ Flujo de Trabajo Rápido

### 1. Iniciar un nuevo proyecto de software
Cuando comiences un nuevo desarrollo (ej. una app Django + React, un pipeline en GCP, o una automatización en n8n):
1. Abre [prompt/prompt_skills_maestras.md](prompt/prompt_skills_maestras.md).
2. Copia el bloque markdown de instrucciones.
3. Pégalo en el archivo de contexto del agente en tu nuevo proyecto:
   * **Antigravity / Gemini CLI**: `agent.md` o `.agents/rules/skills-core.md`
   * **Cursor**: `.cursorrules` o `.cursor/rules/skills-core.mdc`
   * **Claude Code / Desktop**: `CLAUDE.md`
4. El agente leerá automáticamente la matriz de enrutamiento y cargará las skills correspondientes bajo el principio de **Divulgación Progresiva (Progressive Disclosure)**.

### 2. Actualizar el repositorio con aprendizajes de proyectos finalizados
Cuando concluyas un proyecto y desees consolidar nuevas lecciones aprendidas o herramientas:
1. Abre [prompt/prompt_nuevos_skills.md](prompt/prompt_nuevos_skills.md).
2. Proporciona la ruta de la carpeta con las nuevas skills al modelo de IA.
3. El prompt guiará al agente para deduplicar, anonimizar, enriquecer las skills existentes y actualizar los índices maestros automáticamente.

---

## 🛡️ Principios Canónicos

* **Cero Hardcoding**: Ningún skill asume dominios, clientes, IPs ni credenciales reales. Todo parámetro variable se abstrae con `# CONFIG:`.
* **Enfoque en Antipatrones de Batalla**: Cada skill documenta explícitamente el código erróneo de producción (❌ MAL) y su solución (✅ BIEN).
* **Defensa en Profundidad**: La autoridad y validación criptográfica reside en el backend; el cliente es solo interfaz y experiencia de usuario.
* **Idempotencia y Resiliencia**: Operaciones de datos, migraciones y reintentos diseñados para tolerar fallas de red y reejecuciones seguras.

---

## 🚀 Inicialización y Sincronización con GitHub

Para enlazar este directorio local con tu repositorio remoto de GitHub:

```bash
# 1. Inicializar repositorio git local (si aún no lo has hecho)
git init

# 2. Agregar los archivos al área de preparación
git add .

# 3. Crear el commit inicial
git commit -m "feat: inicializacion de ai-skills-core y prompts maestros de gobernanza"

# 4. Configurar la rama principal y el origen remoto
git branch -M main
git remote add origin https://github.com/TU-USUARIO/TU-REPOSITORIO-SKILLS.git

# 5. Subir a GitHub
git push -u origin main
```

---

## 👤 Autor

**Javier Esquivel**  
*Desarrollador Fullstack*  
Especialista en Arquitectura de Software, Ecosistemas Cloud y Soluciones Asistidas por IA.

---

*Mantenido como infraestructura viva de ingeniería de software asistida por IA.*
