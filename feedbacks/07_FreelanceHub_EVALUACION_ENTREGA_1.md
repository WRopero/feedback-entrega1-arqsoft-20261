# Evaluación Técnica - FreelanceHub

---

## Estado Actual del Proyecto

El repositorio contiene únicamente el esqueleto generado por `django-admin startproject`. No hay ninguna app de negocio implementada: sin modelos, sin vistas, sin URLs propias, sin templates, sin tests. Solo existe la configuración base de Django más un `README.md` y `requirements.txt`.

Esto significa que la **primera entrega no cumple con los requisitos mínimos** de implementación. La nota se ajusta a 3.0 por política del curso, pero es fundamental que la segunda entrega muestre avance real y sustancial.

---

## Lo que Debe Implementarse para la Segunda Entrega

A continuación se detalla, de forma exhaustiva, lo que se espera de un marketplace de servicios freelance y qué debe estar listo para la próxima entrega.

---

### 1. Modelos de Dominio (OBLIGATORIO)

El corazón del proyecto es el modelado del dominio. Deben existir al menos:

```python
# accounts/models.py
from django.contrib.auth.models import AbstractUser
from django.db import models

class User(AbstractUser):
    class Role(models.TextChoices):
        CLIENT     = 'client',     'Cliente'
        FREELANCER = 'freelancer', 'Freelancer'

    role = models.CharField(max_length=20, choices=Role.choices)
    bio  = models.TextField(blank=True)

class FreelancerProfile(models.Model):
    user       = models.OneToOneField(User, on_delete=models.CASCADE, related_name='freelancer_profile')
    skills     = models.ManyToManyField('Skill', blank=True)
    hourly_rate = models.DecimalField(max_digits=8, decimal_places=2, null=True, blank=True)
    portfolio_url = models.URLField(blank=True)

class Skill(models.Model):
    name = models.CharField(max_length=100, unique=True)
    def __str__(self): return self.name
```

```python
# services/models.py
import uuid
from django.db import models
from django.conf import settings

class ServiceListing(models.Model):
    class Status(models.TextChoices):
        DRAFT    = 'draft',    'Borrador'
        ACTIVE   = 'active',   'Activo'
        PAUSED   = 'paused',   'Pausado'
        ARCHIVED = 'archived', 'Archivado'

    id          = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
    freelancer  = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.CASCADE, related_name='listings')
    title       = models.CharField(max_length=200)
    description = models.TextField()
    price       = models.DecimalField(max_digits=10, decimal_places=2)
    status      = models.CharField(max_length=20, choices=Status.choices, default=Status.DRAFT)
    created_at  = models.DateTimeField(auto_now_add=True)
    updated_at  = models.DateTimeField(auto_now=True)

class ServiceRequest(models.Model):
    class Status(models.TextChoices):
        PENDING   = 'pending',   'Pendiente'
        ACCEPTED  = 'accepted',  'Aceptado'
        REJECTED  = 'rejected',  'Rechazado'
        COMPLETED = 'completed', 'Completado'
        CANCELLED = 'cancelled', 'Cancelado'

    id        = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
    listing   = models.ForeignKey(ServiceListing, on_delete=models.PROTECT, related_name='requests')
    client    = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.CASCADE, related_name='service_requests')
    message   = models.TextField()
    status    = models.CharField(max_length=20, choices=Status.choices, default=Status.PENDING)
    created_at = models.DateTimeField(auto_now_add=True)
```

---

### 2. Autenticación y Registro (OBLIGATORIO)

Se debe implementar registro y login diferenciado por rol usando JWT:

```python
# accounts/serializers.py
from rest_framework import serializers
from django.contrib.auth import get_user_model

User = get_user_model()

class RegisterSerializer(serializers.ModelSerializer):
    password = serializers.CharField(write_only=True, min_length=8)
    role     = serializers.ChoiceField(choices=User.Role.choices)

    class Meta:
        model  = User
        fields = ['email', 'password', 'first_name', 'last_name', 'role']

    def create(self, validated_data):
        return User.objects.create_user(**validated_data)
```

**No usar `@csrf_exempt` en endpoints de autenticación.** Usar `djangorestframework-simplejwt` correctamente configurado.

---

### 3. Vistas y Endpoints REST (OBLIGATORIO)

Mínimo estos endpoints deben existir y funcionar:

| Método | URL | Descripción |
|--------|-----|-------------|
| POST | `/api/auth/register/` | Registro de usuario |
| POST | `/api/auth/login/` | Login → retorna JWT |
| GET | `/api/listings/` | Listado de servicios activos |
| POST | `/api/listings/` | Crear servicio (solo freelancers) |
| GET | `/api/listings/{id}/` | Detalle de un servicio |
| PUT/PATCH | `/api/listings/{id}/` | Editar servicio (solo dueño) |
| POST | `/api/listings/{id}/request/` | Solicitar un servicio (solo clientes) |
| GET | `/api/my/requests/` | Mis solicitudes (como cliente o freelancer) |

---

### 4. Capa de Servicios (OBLIGATORIO para esta asignatura)

La lógica de negocio no debe vivir en las vistas. Crear `services/services.py`:

```python
from django.db import transaction
from .models import ServiceRequest, ServiceListing

class ServiceRequestService:
    @staticmethod
    @transaction.atomic
    def accept_request(request_obj, acting_user):
        if request_obj.listing.freelancer != acting_user:
            raise PermissionError("Solo el freelancer puede aceptar solicitudes.")
        if request_obj.status != ServiceRequest.Status.PENDING:
            raise ValueError("Solo se pueden aceptar solicitudes pendientes.")
        request_obj.status = ServiceRequest.Status.ACCEPTED
        request_obj.save(update_fields=['status'])
        return request_obj

    @staticmethod
    @transaction.atomic
    def reject_request(request_obj, acting_user):
        if request_obj.listing.freelancer != acting_user:
            raise PermissionError("Solo el freelancer puede rechazar solicitudes.")
        request_obj.status = ServiceRequest.Status.REJECTED
        request_obj.save(update_fields=['status'])
        return request_obj
```

---

### 5. Docker (OBLIGATORIO)

Sin Docker no se puede evaluar el proyecto en la máquina del profesor.

```dockerfile
# Dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 8000
CMD ["python", "manage.py", "runserver", "0.0.0.0:8000"]
```

```yaml
# docker-compose.yml
services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: freelancehub_db
      POSTGRES_USER: freelancehub_user
      POSTGRES_PASSWORD: freelancehub_pass
    volumes:
      - postgres_data:/var/lib/postgresql/data

  web:
    build: .
    command: python manage.py runserver 0.0.0.0:8000
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=postgresql://freelancehub_user:freelancehub_pass@db:5432/freelancehub_db
      - DJANGO_SECRET_KEY=change-me-in-production
      - DEBUG=True
    depends_on:
      - db

volumes:
  postgres_data:
```

---

### 6. Seguridad — Errores Críticos que Corregir

- **`SECRET_KEY` hardcodeada**: La clave secreta actual (`django-insecure-me#0qlbo*ez...`) está en el código fuente. Moverla a variable de entorno **inmediatamente**.
- **`DEBUG = True` hardcodeado**: Debe leerse de variable de entorno.
- **`ALLOWED_HOSTS = []`**: Debe configurarse correctamente.

```python
# settings.py — versión correcta
import os
SECRET_KEY  = os.environ['DJANGO_SECRET_KEY']
DEBUG       = os.getenv('DEBUG', 'False') == 'True'
ALLOWED_HOSTS = os.getenv('ALLOWED_HOSTS', 'localhost').split(',')
```

---

### 7. Tests (OBLIGATORIO para demostrar correctitud)

Al menos 5 tests que cubran flujos críticos:

```python
class ServiceRequestTests(TestCase):
    def test_cliente_puede_solicitar_servicio(self): ...
    def test_freelancer_no_puede_solicitar_su_propio_servicio(self): ...
    def test_solo_dueno_puede_aceptar_solicitud(self): ...
    def test_solicitud_aceptada_no_se_puede_aceptar_de_nuevo(self): ...
    def test_listado_solo_muestra_servicios_activos(self): ...
```

---

### 8. Buenas Prácticas de Git

El proyecto debe mostrar historial de trabajo real:

- Un commit por feature o fix, no un commit masivo al final
- Mensajes descriptivos: `feat(listings): agregar endpoint de creación de servicio`
- Nunca subir `SECRET_KEY`, `.env` ni `__pycache__` al repositorio

---

## Resumen de Prioridades

| Prioridad | Tarea |
|:---------:|-------|
| 🔴 Crítico | Modelos de dominio (User, ServiceListing, ServiceRequest) |
| 🔴 Crítico | Docker funcional |
| 🔴 Crítico | Quitar SECRET_KEY del código |
| 🟠 Alto | Endpoints REST básicos |
| 🟠 Alto | Capa de servicios |
| 🟡 Medio | Tests |
| 🟡 Medio | README con instrucciones de ejecución |

---

*La segunda entrega debe demostrar una aplicación funcional corriendo en Docker con al menos dos flujos de usuario completos.*
