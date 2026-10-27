---
id: "security-hardening"
version: "1.0.0"
domain: "Architecture"
technologies:
  - "JWT (JSON Web Tokens)"
  - "CORS"
  - "OWASP Top 10"
  - "Django Security"
  - "Google Identity Services (GIS)"
  - "HMAC"
triggers:
  - "Implementar autenticación en cualquier API o aplicación"
  - "Endpoint HTTP expuesto públicamente"
  - "Manejo de datos sensibles de usuarios"
  - "CORS configurado incorrectamente"
  - "Implementar autenticación con Google (GIS/OAuth)"
  - "Diseñar autenticación stateless serverless"
related_skills:
  - "django-drf-patterns"
  - "gcp-core-services"
  - "error-handling-patterns"
source_generation: "core"
---

# Security Hardening — JWT, CORS, OWASP, GIS Auth y Autenticación Serverless

> **TL;DR para IA:** Carga esta skill antes de implementar cualquier endpoint público,
> sistema de autenticación o manejo de datos sensibles. Previene las vulnerabilidades más comunes.

---

## 1. Antipatrones Críticos

### ❌ Antipatrón 1: Validar JWT solo en el cliente
```javascript
// ❌ MAL — El cliente no puede ser árbitro de su propia autenticidad
const payload = JSON.parse(atob(token.split('.')[1]));  // Solo decodifica, NO verifica
if (payload.rol === 'admin') { /* acceso garantizado sin verificación */ }
```
**Consecuencia**: Cualquiera puede forjar un JWT modificando el payload en base64.

### ❌ Antipatrón 2: CORS con wildcard en endpoints con credenciales
```python
# ❌ MAL — No puede combinarse Access-Control-Allow-Origin: * con credenciales
CORS_ALLOW_ALL_ORIGINS = True       # + 
CORS_ALLOW_CREDENTIALS = True       # = ERROR de seguridad y de navegador
```

### ❌ Antipatrón 3: Almacenar JWT en localStorage
```javascript
// ❌ MAL — localStorage es accesible por cualquier script XSS
localStorage.setItem('access_token', jwtToken);
```
**Consecuencia**: Un script XSS inyectado roba el token y tiene acceso indefinido.

### ❌ Antipatrón 4: Token con duración indefinida o muy larga
```python
# ❌ MAL — Un token comprometido da acceso para siempre
ACCESS_TOKEN_LIFETIME = timedelta(days=365)
```

---

## 2. Reglas de Oro

1. **La autoridad es siempre el backend**: El cliente presenta credenciales; el servidor las verifica criptográficamente
2. **JWT en httpOnly cookies** (web) o almacenamiento seguro del OS (mobile); nunca localStorage
3. **CORS específico**: `CORS_ALLOWED_ORIGINS` con lista blanca explícita, nunca wildcard con credenciales
4. **Tokens de corta duración**: Access token ≤ 15 min; refresh token con rotación
5. **Validar y sanitizar toda entrada externa**: Incluyendo headers, query params y cuerpo de la request
6. **Mínimo privilegio en permisos**: Los tokens solo contienen los claims necesarios

---

## 3. JWT en Django REST Framework

```python
# config/settings.py — CONFIG: Ajustar tiempos según política de seguridad del proyecto
from datetime import timedelta

SIMPLE_JWT = {
    "ACCESS_TOKEN_LIFETIME": timedelta(minutes=15),     # CONFIG: 5-30 min para producción
    "REFRESH_TOKEN_LIFETIME": timedelta(days=7),        # CONFIG: 1-30 días según riesgo
    "ROTATE_REFRESH_TOKENS": True,                      # Rotar refresh token en cada uso
    "BLACKLIST_AFTER_ROTATION": True,                   # Invalidar tokens anteriores
    "ALGORITHM": "HS256",
    "SIGNING_KEY": env("DJANGO_SECRET_KEY"),            # Desde variable de entorno
    "AUTH_HEADER_TYPES": ("Bearer",),
    "USER_ID_FIELD": "id",
    "USER_ID_CLAIM": "user_id",
    # Incluir claims mínimos necesarios:
    "TOKEN_OBTAIN_SERIALIZER": "apps.auth.serializers.CustomTokenSerializer",
}
```

```python
# apps/auth/serializers.py — Claims mínimos en el JWT
from rest_framework_simplejwt.serializers import TokenObtainPairSerializer

class CustomTokenSerializer(TokenObtainPairSerializer):
    @classmethod
    def get_token(cls, user):
        token = super().get_token(user)
        # CONFIG: agregar solo los claims necesarios para el negocio
        token["email"] = user.email
        token["rol"] = user.perfil.rol  # Solo el rol, no el objeto completo
        return token
```

---

## 4. CORS — Configuración Segura en Django

```python
# config/settings.py
INSTALLED_APPS = [
    ...
    "corsheaders",
]

MIDDLEWARE = [
    "corsheaders.middleware.CorsMiddleware",  # ← Debe ir PRIMERO en MIDDLEWARE
    "django.middleware.common.CommonMiddleware",
    ...
]

# ✅ BIEN — Lista blanca explícita
CORS_ALLOWED_ORIGINS = [
    "https://mi-app.com",                    # CONFIG: dominio de producción
    "https://staging.mi-app.com",            # CONFIG: dominio de staging
    # "http://localhost:5173",               # Solo en desarrollo local (comentar en prod)
]
CORS_ALLOW_CREDENTIALS = True
CORS_ALLOW_METHODS = ["GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"]
CORS_ALLOW_HEADERS = ["accept", "authorization", "content-type", "x-csrftoken"]
```

---

## 5. Autenticación con Google Identity Services (GIS) — Validada en Backend

> Heredado y consolidado de `Skills_g1/serverless-stateless-auth` y `Skills_g2/google-identity-gis-auth`

```
┌─────────┐   1. Iniciar login    ┌─────────────┐
│ Cliente │ ──────────────────── ▶│ Google GIS  │
│  (SPA)  │ ◀─────────────────── │             │
│         │   2. id_token (JWT)   └─────────────┘
│         │
│         │   3. POST /auth/google { id_token }
│         │ ──────────────────────────────────▶ ┌──────────┐
│         │                                     │ Backend  │
│         │                                     │          │
│         │                                     │ 4. Verif │
│         │                                     │ en Google│
│         │   5. { session_token, user }        │ API      │
│         │ ◀────────────────────────────────── └──────────┘
└─────────┘
```

```python
# apps/auth/views.py — Verificación server-side del id_token de Google
import httpx
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status
from rest_framework.permissions import AllowAny

GOOGLE_TOKEN_INFO_URL = "https://oauth2.googleapis.com/tokeninfo"
GOOGLE_CLIENT_ID = env("GOOGLE_OAUTH_CLIENT_ID")     # CONFIG: desde .env

USUARIOS_PERMITIDOS: set[str] = {                    # CONFIG: allowlist de emails
    # Cargar desde base de datos o variable de entorno en producción
}

class GoogleAuthView(APIView):
    permission_classes = [AllowAny]
    
    def post(self, request):
        id_token = request.data.get("id_token")
        if not id_token:
            return Response({"error": "id_token requerido"}, status=400)
        
        # 1. Verificar el token contra la API de Google (nunca decodificar solo en cliente)
        try:
            response = httpx.get(
                GOOGLE_TOKEN_INFO_URL,
                params={"id_token": id_token},
                timeout=10,
            )
            response.raise_for_status()
            claims = response.json()
        except (httpx.HTTPError, Exception):
            return Response({"error": "Token inválido"}, status=401)
        
        # 2. Verificar que el token fue emitido para NUESTRA app
        if claims.get("aud") != GOOGLE_CLIENT_ID:
            return Response({"error": "Token no autorizado para esta aplicación"}, status=401)
        
        # 3. Verificar que el email está verificado por Google
        if claims.get("email_verified") != "true":
            return Response({"error": "Email no verificado"}, status=401)
        
        email = claims.get("email", "").lower()
        
        # 4. Verificar allowlist (si aplica)
        # if email not in USUARIOS_PERMITIDOS:
        #     return Response({"error": "Acceso no autorizado"}, status=403)
        
        # 5. Crear o recuperar usuario y emitir token de sesión propio
        from django.contrib.auth import get_user_model
        from rest_framework_simplejwt.tokens import RefreshToken
        
        User = get_user_model()
        usuario, _ = User.objects.get_or_create(
            email=email,
            defaults={"username": email, "first_name": claims.get("given_name", "")}
        )
        
        refresh = RefreshToken.for_user(usuario)
        return Response({
            "access": str(refresh.access_token),
            "refresh": str(refresh),
            "usuario": {"email": email, "nombre": claims.get("name")},
        })
```

---

## 6. Autenticación HMAC Serverless (sin base de datos)

> Para arquitecturas GAS/serverless sin backend tradicional

```javascript
// Generación de token firmado con HMAC (Google Apps Script / Node.js)
// CONFIG: Cambiar SECRET_KEY por un valor de al menos 32 caracteres aleatorios
const SECRET_KEY = PropertiesService.getScriptProperties().getProperty('HMAC_SECRET');

function generarTokenAcceso(email) {
  const payload = {
    email: email.toLowerCase(),
    exp: Date.now() + (24 * 60 * 60 * 1000), // CONFIG: TTL en ms (24h)
    iat: Date.now(),
  };
  
  const payloadStr = JSON.stringify(payload);
  const payloadB64 = Utilities.base64Encode(payloadStr);
  
  const firma = Utilities.computeHmacSha256Signature(payloadB64, SECRET_KEY);
  const firmaHex = firma.map(b => ('0' + (b & 0xFF).toString(16)).slice(-2)).join('');
  
  return `${payloadB64}.${firmaHex}`;
}

function verificarToken(token) {
  const partes = token.split('.');
  if (partes.length !== 2) return null;
  
  const [payloadB64, firmaRecibida] = partes;
  
  // Verificar firma criptográfica
  const firmaEsperada = Utilities.computeHmacSha256Signature(payloadB64, SECRET_KEY);
  const firmaEsperadaHex = firmaEsperada.map(b => ('0' + (b & 0xFF).toString(16)).slice(-2)).join('');
  
  if (firmaRecibida !== firmaEsperadaHex) return null; // Firma inválida
  
  const payload = JSON.parse(Utilities.newBlob(Utilities.base64Decode(payloadB64)).getDataAsString());
  
  if (Date.now() > payload.exp) return null; // Token expirado
  
  return payload;
}
```

---

## 7. Checklist de Verificación

### Autenticación
- [ ] JWT verificado criptográficamente en el servidor, nunca solo en el cliente
- [ ] Tokens de acceso con duración máxima de 30 minutos
- [ ] Refresh tokens con rotación activada y blacklist
- [ ] Flujos OAuth con verificación del `aud` claim contra el Client ID propio

### CORS
- [ ] `CORS_ALLOWED_ORIGINS` con lista blanca explícita (sin wildcard con credenciales)
- [ ] `corsheaders.middleware.CorsMiddleware` es el PRIMER middleware en Django
- [ ] Headers CORS mínimos: solo los necesarios para el cliente

### OWASP Basics
- [ ] Toda entrada del usuario validada y sanitizada antes de procesarse
- [ ] Sin concatenación de strings para construir queries SQL (usar ORM o params)
- [ ] Headers de seguridad configurados: `X-Frame-Options`, `X-Content-Type-Options`, `HSTS`
- [ ] Rate limiting en endpoints de login/auth
