---
id: "gcp-core-services"
version: "1.0.0"
domain: "DevOps"
technologies:
  - "Google Cloud Platform"
  - "Cloud Run"
  - "BigQuery"
  - "Cloud Storage (GCS)"
  - "IAM & Service Accounts"
  - "Secret Manager"
  - "Cloud Build"
triggers:
  - "Desplegar una aplicación en Google Cloud Run"
  - "Acceder a BigQuery desde Python o desde una aplicación"
  - "Configurar Service Accounts e IAM en GCP"
  - "Almacenar secretos o credenciales en GCP de forma segura"
  - "Configurar CI/CD con Cloud Build"
  - "Leer o escribir archivos en Cloud Storage"
related_skills:
  - "docker-containers"
  - "security-hardening"
  - "python-project-setup"
source_generation: "core"
---

# GCP Core Services — Cloud Run, BigQuery, IAM y Secret Manager

> **TL;DR para IA:** Carga esta skill para desplegar en Cloud Run, consultar BigQuery,
> gestionar IAM con principio de mínimo privilegio y manejar secretos con Secret Manager.

---

## 1. Antipatrones a Evitar

### ❌ Antipatrón 1: Service Account Keys en archivos JSON
```bash
# ❌ MAL — Exportar y usar credenciales en archivos
gcloud iam service-accounts keys create creds.json --iam-account=mi-sa@proyecto.iam.gserviceaccount.com
export GOOGLE_APPLICATION_CREDENTIALS=./creds.json
```
**Consecuencia**: Las claves pueden filtrarse en git, logs o discos. Difícil rotar. GCP las desactivará automáticamente si detecta filtración.

### ❌ Antipatrón 2: Roles primitivos (Owner/Editor) a Service Accounts
```bash
# ❌ MAL — Otorgar roles demasiado amplios
gcloud projects add-iam-policy-binding mi-proyecto \
  --member="serviceAccount:mi-sa@..." \
  --role="roles/editor"  # ← Acceso a casi todo el proyecto
```

### ❌ Antipatrón 3: Variables sensibles en variables de entorno de Cloud Run sin Secret Manager
```bash
# ❌ MAL — Secret visible en la consola y en los logs de despliegue
gcloud run deploy mi-app \
  --set-env-vars="DATABASE_PASSWORD=mi-password-real"  # ← Visible en texto plano
```

---

## 2. Reglas de Oro

1. **Workload Identity / Service Account sin keys**: En Cloud Run, la SA se asigna al servicio; no se necesitan archivos JSON
2. **Mínimo Privilegio (Least Privilege)**: Roles predefinidos específicos (ej: `roles/bigquery.dataViewer`), nunca `roles/editor`
3. **Secret Manager para secretos**: Contraseñas, tokens y API keys en Secret Manager, no en env vars
4. **Un Service Account por servicio**: No compartir SAs entre múltiples servicios
5. **VPC sin IP pública cuando es posible**: Cloud Run con VPC Connector para acceso a recursos internos
6. **Least Privilege en IAM Conditions**: Restringir por recurso específico, no por proyecto entero

---

## 3. Cloud Run — Despliegue desde Docker

```bash
# CONFIG: Ajustar PROJECT_ID, REGION, SERVICE_NAME según el proyecto

PROJECT_ID="mi-proyecto-gcp"        # CONFIG
REGION="us-central1"                 # CONFIG: elegir región cercana a usuarios
SERVICE_NAME="mi-aplicacion"         # CONFIG
IMAGE="gcr.io/$PROJECT_ID/$SERVICE_NAME"

# 1. Construir y subir imagen
gcloud builds submit --tag "$IMAGE" .

# 2. Desplegar en Cloud Run
gcloud run deploy "$SERVICE_NAME" \
  --image "$IMAGE" \
  --region "$REGION" \
  --platform managed \
  --service-account "mi-sa@$PROJECT_ID.iam.gserviceaccount.com" \
  --set-secrets="DATABASE_URL=database-url:latest,SECRET_KEY=app-secret-key:latest" \
  --set-env-vars="DJANGO_SETTINGS_MODULE=config.settings.production" \
  --memory=512Mi \                   # CONFIG: ajustar según carga
  --cpu=1 \                          # CONFIG: ajustar según carga
  --min-instances=0 \                # CONFIG: 0 para scale-to-zero, >0 para latencia baja
  --max-instances=10 \               # CONFIG: ajustar según tráfico esperado
  --concurrency=80 \                 # CONFIG: cuántos requests simultáneos por instancia
  --no-allow-unauthenticated         # Quitar este flag solo si el endpoint es público
```

---

## 4. Secret Manager — Configuración y Uso

```bash
# Crear secretos
echo -n "postgresql://user:pass@host:5432/db" | \
  gcloud secrets create database-url --data-file=-

echo -n "mi-secret-key-super-segura" | \
  gcloud secrets create app-secret-key --data-file=-

# Otorgar acceso al Service Account del servicio
gcloud secrets add-iam-policy-binding database-url \
  --member="serviceAccount:mi-sa@$PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor"
```

```python
# Leer secretos en Python (cuando no se puede usar env vars de Cloud Run)
from google.cloud import secretmanager

def obtener_secreto(nombre_secreto: str, project_id: str) -> str:
    """
    Obtiene el valor de un secreto desde Secret Manager.
    
    Args:
        nombre_secreto: Nombre del secreto en Secret Manager
        project_id: ID del proyecto GCP
        
    Returns:
        str: Valor del secreto en texto plano
    """
    client = secretmanager.SecretManagerServiceClient()
    nombre = f"projects/{project_id}/secrets/{nombre_secreto}/versions/latest"
    response = client.access_secret_version(request={"name": nombre})
    return response.payload.data.decode("UTF-8")
```

---

## 5. BigQuery — Consultas desde Python

```python
# services/bigquery_client.py
# CONFIG: Ajustar PROJECT_ID y DATASET_ID según el proyecto

from google.cloud import bigquery
from google.oauth2 import service_account
from typing import Iterator

PROJECT_ID = "mi-proyecto-gcp"       # CONFIG: o leer desde ENV
DATASET_ID = "mi_dataset"            # CONFIG

def get_bq_client() -> bigquery.Client:
    """
    Retorna un cliente BigQuery autenticado.
    En Cloud Run, usa las credenciales del Service Account del servicio (sin JSON key).
    En local, usa GOOGLE_APPLICATION_CREDENTIALS o Application Default Credentials.
    """
    return bigquery.Client(project=PROJECT_ID)

def ejecutar_query(
    sql: str,
    parametros: list[bigquery.ScalarQueryParameter] | None = None,
) -> Iterator[dict]:
    """
    Ejecuta una query en BigQuery y retorna los resultados como dicts.
    Usa consultas parametrizadas para prevenir SQL injection.
    
    Args:
        sql: Query SQL con parámetros en formato @nombre_param
        parametros: Lista de ScalarQueryParameter
        
    Yields:
        dict: Una fila del resultado como diccionario
    """
    client = get_bq_client()
    config = bigquery.QueryJobConfig(
        query_parameters=parametros or [],
        use_query_cache=True,         # CONFIG: False en datos muy dinámicos
    )
    job = client.query(sql, job_config=config)
    
    for row in job.result():
        yield dict(row)

# Uso con parámetros:
# resultados = ejecutar_query(
#     "SELECT * FROM `dataset.tabla` WHERE estado = @estado AND fecha >= @fecha",
#     parametros=[
#         bigquery.ScalarQueryParameter("estado", "STRING", "activo"),
#         bigquery.ScalarQueryParameter("fecha", "DATE", "2024-01-01"),
#     ]
# )
```

---

## 6. IAM — Roles Mínimos por Caso de Uso

| Caso de Uso | Role GCP Correcto |
|---|---|
| Leer datos de BigQuery | `roles/bigquery.dataViewer` |
| Ejecutar queries en BigQuery | `roles/bigquery.jobUser` + `dataViewer` |
| Leer/escribir en GCS bucket | `roles/storage.objectAdmin` (en el bucket, no el proyecto) |
| Leer secretos | `roles/secretmanager.secretAccessor` |
| Invocar otro Cloud Run | `roles/run.invoker` |
| Publicar en Pub/Sub | `roles/pubsub.publisher` |
| Leer de Pub/Sub | `roles/pubsub.subscriber` |

---

## 7. Checklist de Verificación

### Seguridad
- [ ] Sin archivos `*.json` de credenciales en el repositorio (verificar con `git log --all -- "*.json"`)
- [ ] Service Accounts con roles específicos, nunca `roles/owner` o `roles/editor`
- [ ] Secretos en Secret Manager, referenciados desde Cloud Run, NO en variables de entorno planas
- [ ] Un Service Account diferente por servicio/entorno (prod vs staging)

### Cloud Run
- [ ] `--no-allow-unauthenticated` en servicios internos
- [ ] Límites de memoria y CPU configurados explícitamente
- [ ] `--min-instances=1` si la latencia de cold start es crítica para el negocio

### BigQuery
- [ ] Queries con parámetros (`@param`) en lugar de concatenación de strings SQL
- [ ] Tablas con particionamiento por fecha en datasets de más de 1GB
- [ ] `use_query_cache=True` para queries analíticas repetitivas
