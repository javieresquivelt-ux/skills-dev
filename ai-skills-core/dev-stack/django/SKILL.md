---
id: "django-drf-patterns"
version: "1.0.0"
domain: "Backend"
technologies:
  - "Django 4.2+"
  - "Django REST Framework"
  - "PostgreSQL"
  - "django-environ"
  - "drf-spectacular"
related_skills:
  - "python-project-setup"
  - "security-hardening"
  - "clean-architecture"
  - "error-handling-patterns"
triggers:
  - "Crear un endpoint REST en Django"
  - "Optimizar queries lentos en Django ORM"
  - "Diseñar serializers para modelos relacionales"
  - "Implementar autenticación JWT en Django REST Framework"
  - "Query N+1 detectado en Django"
source_generation: "core"
---

# Django REST Framework — Patrones, ORM Optimization y Serializers

> **TL;DR para IA:** Carga esta skill al construir cualquier endpoint Django, detectar N+1,
> diseñar serializers o configurar autenticación JWT. Contiene patrones anti-N+1 críticos.

---

## 1. Estructura de Aplicación Django

```
apps/
├── core/                    ← Modelos base, mixins, utilidades compartidas
│   ├── models.py            ← BaseModel con timestamps
│   ├── serializers.py       ← BaseSerializer
│   └── permissions.py       ← Permisos personalizados
├── usuarios/                ← App de dominio de usuarios
│   ├── models.py
│   ├── serializers.py
│   ├── views.py
│   ├── urls.py
│   └── tests/
└── <nombre-dominio>/        ← Una app por dominio de negocio
```

---

## 2. Antipatrones a Evitar

### ❌ Antipatrón 1: El Query N+1 (el más costoso)
```python
# ❌ MAL — Genera N+1 queries: 1 para pedidos + N para cada cliente
pedidos = Pedido.objects.all()
for pedido in pedidos:
    print(pedido.cliente.nombre)  # ← Query adicional por cada iteración
```
**Consecuencia**: Con 1000 pedidos = 1001 queries. Degrada el performance exponencialmente.

### ❌ Antipatrón 2: Lógica de negocio en serializers o views
```python
# ❌ MAL — El serializer toma decisiones de negocio
class PedidoSerializer(serializers.ModelSerializer):
    def create(self, validated_data):
        if validated_data['total'] > 10000:
            enviar_email_aprobacion()   # ← No pertenece aquí
        return super().create(validated_data)
```

### ❌ Antipatrón 3: Settings en producción sin variables de entorno
```python
# ❌ MAL — Credenciales hardcodeadas o DEBUG=True en producción
DEBUG = True
DATABASES = {'default': {'PASSWORD': 'mi-password-real'}}
```

---

## 3. Reglas de Oro

1. **Siempre `select_related` / `prefetch_related`**: Antes de iterar un queryset con FK/M2M
2. **Lógica en Services, no en Views/Serializers**: Views = entrada/salida HTTP; Services = negocio
3. **BaseModel con timestamps**: Todos los modelos heredan de un BaseModel con `created_at`, `updated_at`
4. **Serializers tipados**: Usar `serializers.Serializer` con campos explícitos para inputs críticos
5. **`only()` y `defer()`**: Seleccionar solo los campos necesarios en queries de lectura masiva
6. **Paginación obligatoria**: Ningún endpoint devuelve listas sin paginación

---

## 4. BaseModel y Configuración de Settings

```python
# apps/core/models.py
import uuid
from django.db import models

class BaseModel(models.Model):
    """Modelo base con campos de auditoría para todos los modelos del sistema."""
    id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    is_active = models.BooleanField(default=True)

    class Meta:
        abstract = True
        ordering = ["-created_at"]
```

```python
# config/settings.py — CONFIG: Usar django-environ para todas las variables
import environ

env = environ.Env(DEBUG=(bool, False))
environ.Env.read_env()

SECRET_KEY = env("DJANGO_SECRET_KEY")               # CONFIG: obligatorio en .env
DEBUG = env("DEBUG")
ALLOWED_HOSTS = env.list("ALLOWED_HOSTS", default=["localhost"])

DATABASES = {"default": env.db("DATABASE_URL")}     # CONFIG: formato postgres://user:pass@host/db

REST_FRAMEWORK = {
    "DEFAULT_AUTHENTICATION_CLASSES": [
        "rest_framework_simplejwt.authentication.JWTAuthentication",
    ],
    "DEFAULT_PERMISSION_CLASSES": [
        "rest_framework.permissions.IsAuthenticated",
    ],
    "DEFAULT_PAGINATION_CLASS": "rest_framework.pagination.PageNumberPagination",
    "PAGE_SIZE": 20,                                 # CONFIG: ajustar según caso de uso
    "DEFAULT_SCHEMA_CLASS": "drf_spectacular.openapi.AutoSchema",
}
```

---

## 5. Patrón ViewSet + Service Layer

```python
# apps/pedidos/services.py — Lógica de negocio aislada
from django.db import transaction
from .models import Pedido, LineaPedido

class PedidoService:
    @staticmethod
    @transaction.atomic
    def crear_pedido(usuario_id: str, lineas_data: list[dict]) -> Pedido:
        """
        Crea un pedido con sus líneas de forma atómica.
        
        Args:
            usuario_id: UUID del usuario propietario
            lineas_data: Lista de dicts con {producto_id, cantidad, precio_unitario}
            
        Returns:
            Pedido: El pedido creado con todas sus relaciones
        """
        pedido = Pedido.objects.create(usuario_id=usuario_id)
        lineas = [
            LineaPedido(pedido=pedido, **linea)
            for linea in lineas_data
        ]
        LineaPedido.objects.bulk_create(lineas)
        return pedido

# apps/pedidos/views.py — Solo orquestación HTTP
from rest_framework import viewsets, status
from rest_framework.response import Response
from .services import PedidoService
from .serializers import PedidoSerializer, CrearPedidoSerializer

class PedidoViewSet(viewsets.ModelViewSet):
    serializer_class = PedidoSerializer
    
    def get_queryset(self):
        # ✅ BIEN — select_related y prefetch_related en un solo lugar
        return (
            Pedido.objects
            .filter(usuario=self.request.user, is_active=True)
            .select_related("usuario")                         # FK directas
            .prefetch_related("lineas__producto")              # M2M / FK inversas
            .only("id", "created_at", "estado", "usuario__email")  # Solo campos necesarios
        )
    
    def create(self, request, *args, **kwargs):
        serializer = CrearPedidoSerializer(data=request.data)
        serializer.is_valid(raise_exception=True)
        pedido = PedidoService.crear_pedido(
            usuario_id=str(request.user.id),
            lineas_data=serializer.validated_data["lineas"],
        )
        return Response(PedidoSerializer(pedido).data, status=status.HTTP_201_CREATED)
```

---

## 6. Serializers: Input vs Output

```python
# apps/pedidos/serializers.py
from rest_framework import serializers
from .models import Pedido, LineaPedido

class LineaPedidoSerializer(serializers.ModelSerializer):
    """Serializer de salida para líneas de pedido."""
    producto_nombre = serializers.CharField(source="producto.nombre", read_only=True)

    class Meta:
        model = LineaPedido
        fields = ["id", "producto_nombre", "cantidad", "precio_unitario", "subtotal"]
        read_only_fields = ["id", "subtotal"]


class PedidoSerializer(serializers.ModelSerializer):
    """Serializer de salida (lectura) de pedidos con líneas anidadas."""
    lineas = LineaPedidoSerializer(many=True, read_only=True)
    usuario_email = serializers.EmailField(source="usuario.email", read_only=True)

    class Meta:
        model = Pedido
        fields = ["id", "usuario_email", "estado", "total", "lineas", "created_at"]


class CrearLineaSerializer(serializers.Serializer):
    """Serializer de entrada para crear una línea."""
    producto_id = serializers.UUIDField()
    cantidad = serializers.IntegerField(min_value=1, max_value=9999)
    precio_unitario = serializers.DecimalField(max_digits=10, decimal_places=2)


class CrearPedidoSerializer(serializers.Serializer):
    """Serializer de entrada (escritura) — Validación estricta."""
    lineas = CrearLineaSerializer(many=True, min_length=1)

    def validate_lineas(self, lineas: list) -> list:
        """Validar que no haya productos duplicados."""
        ids = [str(l["producto_id"]) for l in lineas]
        if len(ids) != len(set(ids)):
            raise serializers.ValidationError("No puede repetir productos en el pedido.")
        return lineas
```

---

## 7. Checklist de Verificación

### Performance
- [ ] Todos los querysets con iteración usan `select_related` o `prefetch_related`
- [ ] Queries de lectura masiva usan `only()` con los campos estrictamente necesarios
- [ ] Escrituras masivas usan `bulk_create` / `bulk_update` (no loops)
- [ ] Endpoints de lista tienen paginación activa

### Seguridad
- [ ] `DEBUG = False` en producción (leído de ENV)
- [ ] `SECRET_KEY` y `DATABASE_URL` leídos de variables de entorno
- [ ] Sin credenciales en código fuente ni en git history
- [ ] Todos los endpoints de escritura validan con un Serializer de entrada separado

### Arquitectura
- [ ] Lógica de negocio en `services.py`, no en views ni serializers
- [ ] Todos los modelos heredan de `BaseModel`
- [ ] `transaction.atomic` en operaciones que modifican múltiples tablas
- [ ] Tests separados: `tests/unit/` para services, `tests/integration/` para endpoints
