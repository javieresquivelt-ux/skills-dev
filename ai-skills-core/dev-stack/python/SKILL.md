---
id: "python-project-setup"
version: "1.0.0"
domain: "Backend"
technologies:
  - "Python 3.11+"
  - "pip / uv"
  - "pyproject.toml"
  - "mypy"
  - "ruff"
  - "pytest"
triggers:
  - "Inicio de un nuevo proyecto Python"
  - "Configurar entorno virtual, linting o tipado estático"
  - "Crear estructura de módulos Python reutilizable"
  - "Configurar CI con tests automatizados en Python"
related_skills:
  - "django-drf-patterns"
  - "etl-pipeline-patterns"
  - "error-handling-patterns"
source_generation: "core"
---

# Python 3.11+ — Setup de Proyecto, Entornos y Estándares

> **TL;DR para IA:** Usa esta skill al iniciar cualquier proyecto Python para configurar
> la estructura de directorios, entorno virtual, tipado, linting y testing correctamente.

---

## 1. Estructura Canónica de Proyecto Python

```
mi-proyecto/
├── pyproject.toml          ← Config centralizada (ruff, mypy, pytest, build)
├── .python-version         ← Versión exacta del runtime (ej: 3.11.9)
├── README.md
├── .env.example            ← Template de variables de entorno (sin valores reales)
├── .gitignore
├── src/
│   └── mi_proyecto/        ← Paquete principal (src layout)
│       ├── __init__.py
│       ├── config.py       ← Configuración centralizada desde ENV
│       ├── models.py
│       └── services/
│           └── __init__.py
└── tests/
    ├── conftest.py
    ├── unit/
    └── integration/
```

---

## 2. Antipatrones a Evitar

### ❌ Antipatrón 1: Credenciales hardcodeadas
```python
# ❌ MAL — Credenciales en código fuente
DATABASE_URL = "postgresql://user:password123@localhost/mydb"
API_KEY = "sk-proj-abc123real"
```
**Consecuencia**: Exposición de secretos en git history, imposible rotar credenciales.

### ❌ Antipatrón 2: Importaciones implícitas con `from module import *`
```python
# ❌ MAL — Contamina el namespace, rompe el análisis estático
from utils import *
```

### ❌ Antipatrón 3: Funciones sin tipado
```python
# ❌ MAL — mypy no puede verificar, los errores aparecen en runtime
def procesar_datos(datos, config):
    return datos[config["campo"]]
```

---

## 3. Reglas de Oro

1. **Src Layout siempre**: Usar `src/nombre_paquete/` para evitar imports ambiguos
2. **pyproject.toml único**: Toda configuración (ruff, mypy, pytest, build) en un solo archivo
3. **Variables de entorno para secretos**: Usar `python-dotenv` o `pydantic-settings` para config
4. **Tipado explícito**: Todas las funciones públicas con type hints; mypy en modo strict
5. **Tests desde el día 1**: Estructura `tests/` creada antes de escribir la primera función
6. **Ruff sobre flake8+black**: Un solo linter/formatter configurable en pyproject.toml

---

## 4. Configuración pyproject.toml (Boilerplate)

```toml
# ===========================================================================
# pyproject.toml — Configuración centralizada del proyecto Python
# CONFIG: Ajustar [project.name], [project.version] y dependencias
# ===========================================================================

[project]
name = "mi-proyecto"              # CONFIG: nombre del proyecto
version = "0.1.0"
description = "Descripción del proyecto"
requires-python = ">=3.11"
dependencies = [
    "pydantic>=2.0",
    "python-dotenv>=1.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=8.0",
    "pytest-cov>=5.0",
    "ruff>=0.4",
    "mypy>=1.10",
]

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.hatch.build.targets.wheel]
packages = ["src/mi_proyecto"]    # CONFIG: ruta al paquete principal

[tool.ruff]
line-length = 88
target-version = "py311"

[tool.ruff.lint]
select = ["E", "F", "W", "I", "N", "UP", "B", "C4", "SIM"]
ignore = ["E501"]

[tool.mypy]
python_version = "3.11"
strict = true
warn_return_any = true
ignore_missing_imports = true

[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = "--cov=src --cov-report=term-missing"
```

---

## 5. Configuración de Variables de Entorno

```python
# src/mi_proyecto/config.py
# ===========================================================================
# CONFIG: Agregar/quitar campos según las necesidades del proyecto
# ===========================================================================
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
        case_sensitive=False,
    )
    
    # CONFIG: Definir los campos necesarios para el proyecto
    database_url: str
    secret_key: str
    debug: bool = False
    allowed_hosts: list[str] = ["localhost"]
    log_level: str = "INFO"

# Singleton de configuración — importar desde aquí en toda la app
settings = Settings()
```

```bash
# .env.example — COMMIT ESTO. Nunca commitear .env con valores reales
DATABASE_URL=postgresql://user:pass@localhost:5432/dbname
SECRET_KEY=cambia-esto-por-un-valor-seguro-en-produccion
DEBUG=false
LOG_LEVEL=INFO
```

---

## 6. Setup del Entorno de Desarrollo

```bash
# Usar uv (más rápido que pip) o pip tradicional
# Con uv (recomendado):
uv venv --python 3.11
source .venv/bin/activate
uv pip install -e ".[dev]"

# Con pip:
python3.11 -m venv .venv
source .venv/bin/activate
pip install -e ".[dev]"

# Verificar configuración
ruff check src/
mypy src/
pytest
```

---

## 7. Checklist de Verificación

### Setup
- [ ] Archivo `.python-version` con versión exacta (ej: `3.11.9`)
- [ ] `pyproject.toml` presente con ruff, mypy y pytest configurados
- [ ] `src/` layout en lugar de raíz plana
- [ ] `.env.example` commiteado, `.env` en `.gitignore`

### Código
- [ ] Todas las funciones públicas tienen type hints
- [ ] `mypy src/` pasa sin errores en modo strict
- [ ] `ruff check src/` pasa sin errores
- [ ] Sin imports `*` en ningún módulo

### Testing
- [ ] `tests/conftest.py` con fixtures base
- [ ] Al menos un test unitario por módulo principal
- [ ] `pytest` ejecuta sin errores desde el directorio raíz
