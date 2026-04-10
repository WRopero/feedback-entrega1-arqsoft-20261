# Evaluación Técnica - StayHome

> **Nota:** El link enviado apuntaba a la wiki del repositorio (`/wiki/Activities`) en lugar del repo principal. La evaluación automatizada no encontró el código. El repositorio correcto es `Horizonte4/stayhome` y contiene una implementación completa y de muy alta calidad.

---

## Análisis del Código Implementado

Este proyecto presenta la arquitectura más madura de la entrega. Demuestra comprensión real de principios SOLID y patrones de diseño aplicados a una plataforma de alquiler y venta de propiedades.

### Arquitectura General

Cinco apps con responsabilidades claramente delimitadas: `users`, `properties`, `transactions`, `comunication` y `core`. Lo más destacable es la adopción consistente del **patrón services/selectors/exceptions** (Django Styleguide):

- `transactions/services.py` — lógica de negocio orquestada (`BookingService`, `PurchaseRequestService`)
- `transactions/selectors.py` — queries de lectura puras (`has_sale_contract`, `get_client_bookings_context`)
- `transactions/exceptions.py` — excepciones de dominio descriptivas

Esta separación es exactamente lo que se espera en aplicaciones Django profesionales.

### Modelos de Datos

El modelado refleja correctamente el dominio:

- `Booking` con estados gestionados por servicio
- `PurchaseRequest` como entidad formal de solicitud de compra
- `Contract` como registro inmutable de venta sellada
- `Property` con `listing_type` (alquiler vs venta) y `active_listing`

La relación `Contract → Property` protege el histórico de ventas. El uso de `select_related` y `prefetch_related` es consistente en todos los selectores.

### Capa de Servicios

`PurchaseRequestService.accept_request` está bien diseñado:

```python
@staticmethod
@transaction.atomic
def accept_request(purchase_request, acting_user):
    PurchaseRequestService._validate_purchasable_property(purchase_request.property)
    # Crea contrato, desactiva listing, rechaza otras solicitudes pendientes
```

El uso de `@transaction.atomic` garantiza que si falla cualquier paso, se deshace todo. Las validaciones previas se encapsulan en métodos privados del servicio.

### Excepciones de Dominio

```python
class OwnerCannotBuyOwnPropertyError(PurchaseRequestError): ...
class DuplicatePurchaseRequestError(PurchaseRequestError): ...
class PropertyAlreadyPurchasedError(PurchaseRequestError): ...
```

Las vistas capturan estas excepciones y retornan HTTP apropiado. Esto mantiene limpia la separación entre dominio y presentación.

### Celery y Tareas Asíncronas

`tasks.py` en múltiples apps indica que el equipo pensó en flujos reales más allá del request/response síncrono.

---

## Áreas de Mejora

**1. Strings mágicos en estados de Booking**

Los estados se usan como strings literales dispersos en lugar de `TextChoices`:

```python
# Actual (frágil)
Booking.objects.filter(status="pending")
booking.status = "approved"

# Correcto
class Status(models.TextChoices):
    PENDING   = 'pending',   'Pendiente'
    APPROVED  = 'approved',  'Aprobado'
    REJECTED  = 'rejected',  'Rechazado'
    CANCELLED = 'cancelled', 'Cancelado'
```

Un typo como `"pendingg"` falla silenciosamente en runtime.

**2. Race condition en `create_booking`**

El chequeo de disponibilidad y la creación de la reserva no están en la misma transacción atómica. Dos usuarios pueden pasar `has_conflict()` simultáneamente antes de que ninguno grabe su reserva:

```python
@staticmethod
@transaction.atomic
def create_booking(property, user, check_in, check_out):
    property = Property.objects.select_for_update().get(pk=property.pk)
    if BookingService.has_conflict(property, check_in, check_out):
        raise BookingConflictError(...)
    return Booking.objects.create(...)
```

**3. Política de cancelación hardcodeada**

```python
limit_date = booking.check_in - timedelta(days=5)  # ← número mágico
```

Mover a configuración: `settings.BOOKING_CANCEL_DAYS_LIMIT = 5`.

**4. Repository de selectores como módulo plano**

Las funciones de `selectors.py` son todas independientes y sin estado. Podrían moverse a managers de queryset en los modelos para hacer el código más expresivo:

```python
# En lugar de: has_sale_contract(property_obj)
property_obj.has_sale_contract()  # método de instancia
```

---

## Buenas Prácticas de Git

89 commits con historial activo y uso de feature branches con pull requests. Es el proyecto con mayor actividad de commits de la entrega.

---

## Requisitos para Entrega 2 (Rúbrica)

### 1. Correcciones de la Parte 1 (10%)

- [ ] Reemplazar strings mágicos por `TextChoices` en estados de Booking
- [ ] Agregar `@transaction.atomic` + `select_for_update` en `create_booking`
- [ ] Mover constante de días de cancelación (5) a `settings.py`

### 2. Diagrama de arquitectura actualizado (5%)

Entregar diagrama que refleje la arquitectura actual del sistema: capas (presentación, servicios, dominio, persistencia), componentes principales, y las interfaces abstractas (DIP). Formato legible (draw.io, Mermaid, PlantUML).

### 3. Servicios implementados (30%)

Ya tienen servicios. Agregar: servicio de notificaciones multicanal, servicio de reportes/estadísticas

Cada servicio debe:
- Estar en un archivo `services.py` dentro de su app
- Encapsular la lógica de negocio (las vistas solo delegan)
- Usar `@transaction.atomic` donde haya operaciones de escritura múltiples

### 4. Inversión de dependencias (15%)

Ya tienen service layer. Crear interfaz para notificaciones:

```python
# notifications/base.py
from abc import ABC, abstractmethod

class NotificationChannel(ABC):
    @abstractmethod
    def send(self, user, subject, message) -> None: ...

# notifications/email_channel.py
class EmailChannel(NotificationChannel):
    def send(self, user, subject, message) -> None:
        send_mail(subject, message, ..., [user.email])

# notifications/push_channel.py
class PushChannel(NotificationChannel):
    def send(self, user, subject, message) -> None:
        # Firebase Cloud Messaging o similar
        ...
```

El `BookingService` inyecta `NotificationChannel` en lugar de hardcodear el canal.

### 5. Pruebas unitarias (10%)

Implementar al menos **dos pruebas unitarias** que verifiquen lógica de negocio:

```python
def test_owner_no_puede_reservar_su_propia_propiedad(self):
    # owner intenta crear booking → ValueError

def test_aceptar_solicitud_compra_rechaza_las_demas_pendientes(self):
    # Crear 3 solicitudes pendientes, aceptar 1 → las otras 2 quedan rejected
```

### 6. Calidad del código y arquitectura (15%)

- Separación clara en capas (vistas → servicios → modelos)
- Sin lógica de negocio en vistas
- Sin imports circulares entre apps
- Sin secretos en el código fuente
- Código limpio, sin archivos generados automáticamente sin contenido

### 7. Despliegue en nube + sistema de dos idiomas (15%)

- **Despliegue**: El proyecto debe estar desplegado y accesible en un servicio cloud (Railway, Render, AWS, GCP, Azure, etc.)
- **Internacionalización (i18n)**: Implementar soporte para **dos idiomas** (español + inglés) — ya tienen i18n parcial implementado, verificar que cubra toda la interfaz

```python
# settings.py
from django.utils.translation import gettext_lazy as _

LANGUAGES = [
    ('es', _('Español')),
    ('en', _('English')),
]
LOCALE_PATHS = [BASE_DIR / 'locale']
USE_I18N = True

MIDDLEWARE = [
    ...
    'django.middleware.locale.LocaleMiddleware',
    ...
]
```
