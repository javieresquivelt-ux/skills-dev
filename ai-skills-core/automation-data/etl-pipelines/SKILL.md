---
id: "etl-pipeline-patterns"
version: "1.0.0"
domain: "Data"
technologies:
  - "Python 3.11+"
  - "Pydantic V2"
  - "DuckDB / Polars / Pandas"
  - "PostgreSQL / BigQuery"
  - "Parquet"
triggers:
  - "Construir un script o job de ingesta y transformación de datos (ETL/ELT)"
  - "Procesar archivos CSV, JSON masivos o eventos de APIs externas"
  - "Validar esquemas de datos entrantes antes de insertarlos en el almacén"
  - "Garantizar idempotencia en cargas masivas (evitar duplicados al reejecutar)"
related_skills:
  - "python-project-setup"
  - "gcp-core-services"
  - "error-handling-patterns"
source_generation: "core"
---

# ETL Pipelines en Python — Ingesta, Validación Estricta y Carga Idempotente

> **TL;DR para IA:** Utiliza esta skill para diseñar pipelines de extracción, validación y carga en Python. Prioriza procesamiento por chunks en memoria constante, esquemas tipados con Pydantic V2 y cargas idempotentes con UPSERT/MERGE.

---

## 1. Fases Canónicas de un Pipeline Robusto

```
Extracción (Chunked / Stream)
   │
   ▼
Validación y Sanitización (Pydantic V2 Schema)
   │
   ├── [Válidos] ─────────────┐
   └── [Inválidos] ──────────┐│
                             │▼
                     Transformación Inmutable
                             │
                             ▼
                     Carga Idempotente (Upsert / Staging + Merge)
                             │
                             ▼
                     Registro de Auditoría y Métricas (Total, OK, Rejected)
```

---

## 2. Antipatrones a Evitar

### ❌ Antipatrón 1: Cargar datasets gigantescos enteros en memoria
```python
# ❌ MAL: Si el CSV tiene 10 GB, causa Out-Of-Memory (OOM) y aborta el proceso
df = pd.read_csv("archivo_gigante.csv")
```

### ❌ Antipatrón 2: Cargas no idempotentes (Append ciego)
```python
# ❌ MAL: Si el pipeline falla al 90% y se reejecuta, duplica todos los registros previos
df.to_sql("transacciones", engine, if_exists="append")
```

### ❌ Antipatrón 3: Tratar errores de datos abortando todo el pipeline
```python
# ❌ MAL: 1 fila corrupta de 1,000,000 invalida 999,999 registros limpios
for fila in filas:
    procesar_o_morir(fila)
```

---

## 3. Reglas de Oro

1. **Idempotencia Garantizada**: Ejecutar el pipeline 1 vez o 10 veces con el mismo input debe producir exactamente el mismo estado final en el destino.
2. **Validación Temprana**: Rechazar o desviar a una *Dead-Letter Table/Queue* las filas corruptas, registrando la razón exacta sin frenar el flujo principal.
3. **Memoria Acotada ($O(1)$)**: Procesar siempre en iteradores, generadores o chunks (ej. lotes de 1,000 a 10,000 registros).
4. **Transformaciones Puras**: Las funciones de transformación reciben un registro/lote y retornan uno nuevo sin mutar el original.

---

## 4. Patrón de Implementación: Pipeline con Validación Pydantic y Chunking

```python
from typing import Generator, Any
from pydantic import BaseModel, Field, EmailStr, ValidationError
import csv

# --- ESQUEMA DE VALIDACIÓN (Pydantic V2) ---
class RegistroClienteRaw(BaseModel):
    id_externo: str = Field(min_length=1)
    email: EmailStr
    monto_consumo: float = Field(ge=0.0)
    fecha_transaccion: str

class RegistroClienteTransformado(BaseModel):
    id_externo: str
    email: str
    monto_consumo_usd: float
    fecha_iso: str

# --- FASE 1: EXTRACCIÓN POR CHUNKS ---
def extraer_lotes_csv(
    filepath: str,
    tamano_lote: int = 1000
) -> Generator[list[dict[str, Any]], None, None]:
    """Generador que cede lotes delimitados de filas para consumo eficiente de memoria."""
    with open(filepath, mode="r", encoding="utf-8") as f:
        reader = csv.DictReader(f)
        lote: list[dict[str, Any]] = []
        for fila in reader:
            lote.append(fila)
            if len(lote) >= tamano_lote:
                yield lote
                lote = []
        if lote:
            yield lote

# --- FASE 2: TRANSFORMACIÓN DEFENSIVA Y DEAD-LETTER ---
def procesar_lote(
    lote_crudo: list[dict[str, Any]]
) -> tuple[list[RegistroClienteTransformado], list[dict[str, Any]]]:
    validos: list[RegistroClienteTransformado] = []
    rechazados: list[dict[str, Any]] = []

    for item in lote_crudo:
        try:
            # Validación estricta
            registro = RegistroClienteRaw.model_validate(item)
            # Transformación de negocio
            validos.append(
                RegistroClienteTransformado(
                    id_externo=registro.id_externo.strip().upper(),
                    email=registro.email.lower(),
                    monto_consumo_usd=round(registro.monto_consumo, 2),
                    fecha_iso=registro.fecha_transaccion,
                )
            )
        except ValidationError as e:
            rechazados.append({"raw": item, "errores": e.errors()})

    return validos, rechazados

# --- FASE 3: CARGA IDEMPOTENTE (Staging + UPSERT) ---
def cargar_lote_idempotente(lote: list[RegistroClienteTransformado], db_conn: Any):
    """
    Ejemplo conceptual de carga usando patrón Staging Table + MERGE / ON CONFLICT.
    """
    # SQL:
    # INSERT INTO clientes (id_externo, email, monto_usd, fecha)
    # VALUES (%s, %s, %s, %s)
    # ON CONFLICT (id_externo) DO UPDATE SET
    #   email = EXCLUDED.email,
    #   monto_usd = EXCLUDED.monto_usd,
    #   updated_at = NOW();
    pass
```

---

## 5. Checklist de Verificación

- [ ] Las transformaciones y validaciones tienen tests unitarios con datos representativos y casos límite
- [ ] La ingesta de archivos no realiza `f.read()` o `read_csv()` completo sin streaming / chunking
- [ ] El destino implementa claves primarias / de negocio únicas y estrategias `MERGE` / `ON CONFLICT`
- [ ] Existe una tabla, archivo o log para los registros inválidos (*Dead-Letter*) con detalle del error
- [ ] El pipeline emite métricas claras al finalizar: registros leídos, insertados, actualizados y rechazados
