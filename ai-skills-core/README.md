# 🧠 ai-skills-core — Repositorio Centralizado de Skills de IA

> **Versión:** 1.0.0 · **Estado:** Activo · **Audiencia:** LLMs, Agentes Autónomos, Arquitectos y Desarrolladores Senior

Este repositorio es la **base de conocimiento canónica** para inicializar, guiar y gobernar cualquier proyecto de software moderno. Cada skill encapsula arquitecturas probadas en producción, lecciones aprendidas, patrones de diseño y advertencias contra antipatrones críticos.

---

## ⚡ Inicio Rápido — Matriz de Enrutamiento por Tipo de Proyecto

> **Para agentes IA**: Examina esta tabla primero. Carga únicamente las skills relevantes al contexto del proyecto activo para no saturar la ventana de contexto.

| Tipo de Proyecto | Skills Obligatorias | Skills Complementarias |
|---|---|---|
| **Django REST API + PostgreSQL** | `django-drf-patterns`, `python-project-setup`, `security-hardening` | `error-handling-patterns`, `clean-architecture` |
| **Django + React (Fullstack SaaS)** | `django-drf-patterns`, `python-project-setup`, `react-component-patterns`, `sass-7-1-architecture`, `security-hardening` | `modular-rbac-spa`, `local-first-hybrid-storage`, `error-handling-patterns` |
| **React SPA + GAS/Sheets Backend** | `react-component-patterns`, `sass-7-1-architecture`, `google-sheets-baas`, `serverless-stateless-auth` | `modular-rbac-spa`, `config-driven-dynamic-ui`, `local-first-hybrid-storage` |
| **React SPA standalone** | `react-component-patterns`, `sass-7-1-architecture`, `modular-rbac-spa` | `local-first-hybrid-storage`, `config-driven-dynamic-ui` |
| **Google Apps Script API pública** | `gas-rest-api-design`, `sheets-as-database`, `serverless-stateless-auth` | `timezone-wallclock-engine`, `gas-node-testing-harness` |
| **Pipeline ETL en Python (GCP)** | `python-project-setup`, `etl-pipeline-patterns`, `gcp-core-services` | `data-engineering-principles`, `error-handling-patterns` |
| **Flujo n8n (Automatización)** | `n8n-workflow-design`, `error-handling-patterns` | `security-hardening` |
| **Despliegue en GCP (Cloud Run)** | `gcp-core-services`, `docker-containers`, `security-hardening` | `clean-architecture` |
| **Despliegue en Easypanel / Hostinger** | `easypanel-deployment`, `hostinger-dns-setup`, `docker-containers` | `security-hardening` |
| **App offline-first** | `local-first-hybrid-storage`, `react-component-patterns` | `error-handling-patterns` |
| **Pair Programming con IA** | `ai-pair-programming-harness` | *(cualquier skill del dominio activo)* |

---

## 🗂️ Taxonomía del Repositorio

```
ai-skills-core/
├── README.md                           ← Índice maestro (este archivo)
├── meta/
│   ├── CONTRIBUTING.md                 ← Estándares de contribución
│   ├── template-skill.md               ← Plantilla SKILL-SPEC-v1
│   └── migration-guide.md              ← Guía para migrar skills heredadas
│
├── dev-stack/
│   ├── python/SKILL.md                 ← Python 3.11+, entornos, tipado, testing
│   ├── react/SKILL.md                  ← React Hooks, Context, patrones
│   ├── react/sass-7-1-architecture/SKILL.md
│   ├── django/SKILL.md                 ← DRF, ORM, serializers, ViewSets
│   └── frontend-core/SKILL.md          ← Config-driven UI, RBAC-SPA, Local-First, GAS
│
├── devops-cloud/
│   ├── gcp/SKILL.md                    ← Cloud Run, BigQuery, IAM, GCS
│   ├── easypanel/SKILL.md              ← Deploy self-hosted Easypanel
│   ├── hostinger/SKILL.md              ← DNS, VPS, SSL Hostinger
│   └── docker/SKILL.md                 ← Dockerfiles, Compose, dual-repo deploy
│
├── automation-data/
│   ├── n8n/SKILL.md                    ← Workflows, webhooks, sub-flows, errores
│   ├── etl-pipelines/SKILL.md          ← ETL Python, ingesta, transformación, carga
│   └── data-engineering/SKILL.md       ← Modelado relacional, BigQuery, SQL optimization
│
└── architecture-patterns/
    ├── clean-architecture/SKILL.md     ← SOLID, capas, contratos
    ├── security/SKILL.md               ← JWT, CORS, OWASP, GIS Auth, HMAC
    └── error-handling/SKILL.md         ← Excepciones tipadas, logging, resiliencia
```

---

## 🛡️ Principios Transversales

| Principio | Descripción |
|---|---|
| **Cero Hardcoding** | Sin marcas, dominios, credenciales ni estructuras rígidas de cliente |
| **Parametrizable** | Todo elemento variable se abstrae como parámetro o variable de entorno |
| **Defensa en Profundidad** | Autoridad y seguridad siempre en el backend |
| **Progressive Disclosure** | El agente carga solo el contexto relevante al problema actual |
| **Agnóstico de Cliente** | Reutilizable en cualquier proyecto sin modificación estructural |

---

## 🔄 Historial de Generaciones

| Generación | Origen | Dominio | Skills |
|---|---|---|---|
| **g1** | `Skills_g1/` | Google Workspace, GAS, Auth serverless | 7 |
| **g2** | `Skills_g2/` | Frontend SPA, RBAC, Sass, Alertas, IA | 8 |
| **core** | `ai-skills-core/` | Fullstack + Cloud + Datos + Automatización | ∞ |

---

## 👤 Autor

**Javier Esquivel**  
*Desarrollador Fullstack*  
Especialista en Arquitectura de Software, Ecosistemas Cloud y Soluciones Asistidas por IA.

---

*Última actualización: 2026-10-27*
