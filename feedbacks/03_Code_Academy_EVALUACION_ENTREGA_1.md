# Evaluación Técnica - Code Academy

---

## Análisis del Código Implementado

He revisado su plataforma de eCommerce para cursos y libros de programación. El proyecto muestra una arquitectura bien pensada con separación de apps Django y modelos complejos, con configuración Docker completa (backend, frontend y base de datos).

### Modelos de Datos

**Archivos revisados:** 
- `app/products/models.py` (266 líneas, 7 modelos)
- `app/orders/models.py` (89 líneas, 2 modelos)

Su modelado de datos es sofisticado y demuestra comprensión avanzada de Django:

**Product**: Modelo polimórfico que maneja tanto cursos como libros usando un campo `type`. Esto es una decisión de diseño inteligente que evita duplicación de código.

```python
TYPE_COURSE = 'course'
TYPE_BOOK = 'book'
TYPE_CHOICES = [
    (TYPE_COURSE, 'Curso'),
    (TYPE_BOOK, 'Libro'),
]
```

**Fortalezas identificadas:**

1. **Validadores en campos numéricos**: Uso correcto de `MinValueValidator` y `MaxValueValidator` para precio y rating.

2. **Relaciones bien diseñadas**: 
   - `Category` → `Product` (muchos a uno con PROTECT)
   - `Product` → `Chapter` (uno a muchos con CASCADE)
   - `limit_choices_to` para restringir relaciones solo a cursos o libros

3. **Soft delete**: Campo `is_active` para ocultar productos sin eliminarlos de la base de datos.

4. **Timestamps automáticos**: `auto_now_add` y `auto_now` para auditoría.

5. **Modelos avanzados implementados**:
   - `BookDownload`: Control de descargas con límite (3 descargas máximo)
   - `CourseProgress`: Tracking de progreso con ManyToMany a capítulos completados
   - `CourseCertificate`: Generación de certificados PDF al completar cursos

6. **Método de negocio en modelo**:

```python
def recalculate_progress(self):
    total = self.product.chapters.count()
    completed = self.completed_chapters.count()
    self.progress_percentage = 0 if total == 0 else int((completed / total) * 100)
    self.save(update_fields=['progress_percentage', 'updated_at'])
```

**Áreas de mejora:**

1. **Falta validación de coherencia entre tipo y campos específicos**. Un libro no debería tener `duration` y un curso no debería tener `pages`:

```python
from django.core.exceptions import ValidationError

class Product(models.Model):
    # ... campos existentes ...
    
    def clean(self):
        if self.type == self.TYPE_COURSE and self.pages:
            raise ValidationError('Un curso no puede tener páginas')
        if self.type == self.TYPE_BOOK and self.duration:
            raise ValidationError('Un libro no puede tener duración')
        if self.type == self.TYPE_BOOK and not self.book_file:
            raise ValidationError('Un libro debe tener archivo PDF')
```

2. **El modelo Order tiene snapshot pero podría mejorar**:

```python
# Agregar campos para auditoría completa
class Order(models.Model):
    # ... campos existentes ...
    user_email = models.EmailField()  # Snapshot del email del usuario
    user_name = models.CharField(max_length=255)  # Snapshot del nombre
    ip_address = models.GenericIPAddressField(null=True)  # Para seguridad
```

3. **Falta índice compuesto para consultas frecuentes**:

```python
class Product(models.Model):
    class Meta:
        indexes = [
            models.Index(fields=['type', 'is_active', '-created_at']),
            models.Index(fields=['category', 'is_active']),
        ]
```

---

### Vistas y Lógica de Negocio

**Archivo:** `app/products/views.py` (216 líneas)

Implementaron vistas basadas en clases de DRF con filtros avanzados.

**Aspectos destacados:**

1. **Uso de DjangoFilterBackend**: Filtros por tipo, nivel, idioma, categoría.

2. **SearchFilter**: Búsqueda en título, autor y descripción.

3. **OrderingFilter**: Ordenamiento por precio, rating, fecha.

4. **Optimización de queries**:

```python
def get_queryset(self):
    return (
        Product.objects
        .filter(is_active=True)
        .select_related('category')
        .prefetch_related('chapters', 'table_of_contents')
    )
```

5. **Anotaciones para evitar queries N+1**:

```python
Category.objects
    .annotate(active_products=Count('products', filter=Q(products__is_active=True)))
    .filter(active_products__gt=0)
```

**Problemas encontrados:**

1. **No hay control de permisos en vistas de descarga**. Cualquiera podría acceder si conoce la URL:

```python
class BookDownloadView(APIView):
    permission_classes = [IsAuthenticated]  # DEBE estar autenticado
    
    def get(self, request, product_id):
        product = get_object_or_404(Product, id=product_id, type=Product.TYPE_BOOK)
        
        # Verificar que el usuario compró el libro
        has_purchased = Order.objects.filter(
            user=request.user,
            status=Order.STATUS_COMPLETED,
            items__product=product
        ).exists()
        
        if not has_purchased:
            return Response(
                {'error': 'Debes comprar este libro para descargarlo'},
                status=status.HTTP_403_FORBIDDEN
            )
        
        # Verificar límite de descargas
        download, created = BookDownload.objects.get_or_create(
            user=request.user,
            product=product
        )
        
        if download.download_count >= download.max_downloads:
            return Response(
                {'error': f'Has alcanzado el límite de {download.max_downloads} descargas'},
                status=status.HTTP_403_FORBIDDEN
            )
        
        # Incrementar contador
        download.download_count += 1
        download.last_downloaded_at = timezone.now()
        download.save()
        
        # Servir archivo
        return FileResponse(
            product.book_file.open('rb'),
            as_attachment=True,
            filename=f"{product.title}.pdf"
        )
```

3. **Integración con Stripe mencionada pero no visible en el código revisado**. Esto debería estar en `orders/views.py`.

---

### Integración con Stripe

El README menciona integración con Stripe para pagos, lo cual es excelente. Sin embargo, necesitan asegurar:

**Webhook de Stripe para actualizar estado de orden**:

```python
import stripe
from django.conf import settings
from django.views.decorators.csrf import csrf_exempt

stripe.api_key = settings.STRIPE_SECRET_KEY

@csrf_exempt
def stripe_webhook(request):
    payload = request.body
    sig_header = request.META['HTTP_STRIPE_SIGNATURE']
    
    try:
        event = stripe.Webhook.construct_event(
            payload, sig_header, settings.STRIPE_WEBHOOK_SECRET
        )
    except ValueError:
        return HttpResponse(status=400)
    except stripe.error.SignatureVerificationError:
        return HttpResponse(status=400)
    
    if event['type'] == 'payment_intent.succeeded':
        payment_intent = event['data']['object']
        
        # Actualizar orden
        order = Order.objects.get(payment_reference=payment_intent['id'])
        order.status = Order.STATUS_COMPLETED
        order.save()
        
        # Enviar email de confirmación
        send_order_confirmation_email(order)
    
    return HttpResponse(status=200)
```

---

### Arquitectura de Apps

Excelente separación en apps Django:
- `core`: Configuración base
- `users`: Autenticación y usuarios
- `products`: Catálogo de productos
- `orders`: Órdenes y pagos

Esto facilita el mantenimiento y escalabilidad.

---

### Patrones y Principios SOLID

**Lo que funciona bien:**
- Separación en apps por dominio (`products`, `orders`, `users`) — SRP a nivel de módulo
- `products/services.py` para generación de certificados — lógica fuera de vistas
- `BookDownload` con control de descargas máximas — buen modelado de regla de negocio
- `CourseProgress.recalculate_progress()` como método de dominio — lógica donde pertenece
- Integración real con Stripe incluyendo webhook asíncrono
- Uso correcto de `django-filters` para filtrado declarativo

**Problemas arquitectónicos:**

1. **`Product` con campos de tipo exclusivo — viola cohesión.** `duration` (solo cursos) y `pages` (solo libros) coexisten en la misma tabla. Un libro siempre tiene `duration=''` y un curso `pages=None`. Considerar herencia abstracta o al menos validación a nivel de modelo:

```python
def clean(self):
    if self.type == self.TYPE_BOOK and self.duration:
        raise ValidationError("Los libros no tienen duración")
    if self.type == self.TYPE_COURSE and self.pages:
        raise ValidationError("Los cursos no tienen páginas")
```

2. **`rating` desnormalizado sin mecanismo de actualización.** Si alguien crea una reseña sin pasar por el endpoint, el rating del producto queda stale. Debe calcularse con `Avg('reviews__rating')` o actualizarse vía signal/servicio.

3. **`_grant_user_access_for_order` y `_mark_order_completed` son funciones sueltas en `orders/views.py`** — son lógica de dominio que debería vivir en un `OrderService` o como métodos del modelo `Order`.

4. **Inconsistencia de indentación en `orders/views.py`** — usa tabs en lugar de 4 espacios (PEP8). Esto puede causar `IndentationError` al mezclar con otros archivos.

5. **`category__name` expuesto en filtros de API** — expone la estructura interna del ORM. Usar un `FilterSet` personalizado que mapee `?category=Python` internamente a `category__name`.


---

## Requisitos para Entrega 2 (Rúbrica)

### 1. Correcciones de la Parte 1 (10%)

- [ ] Resolver inconsistencia de indentación (tabs vs spaces) en `orders/views.py`
- [ ] Calcular `rating` dinámicamente en vez de campo desnormalizado
- [ ] Mover `_grant_user_access_for_order` y `_mark_order_completed` a un `OrderService`
- [ ] Usar FilterSet personalizado en lugar de exponer `category__name` directamente

### 2. Diagrama de arquitectura actualizado (5%)

Entregar diagrama que refleje la arquitectura actual del sistema: capas (presentación, servicios, dominio, persistencia), componentes principales, y las interfaces abstractas (DIP). Formato legible (draw.io, Mermaid, PlantUML).

### 3. Servicios implementados (30%)

Servicio de órdenes (extraer lógica de vistas), servicio de progreso de curso, servicio de generación de certificados

Cada servicio debe:
- Estar en un archivo `services.py` dentro de su app
- Encapsular la lógica de negocio (las vistas solo delegan)
- Usar `@transaction.atomic` donde haya operaciones de escritura múltiples

### 4. Inversión de dependencias (15%)

Ya tienen una base con `services.py`. Crear una interfaz para el servicio de notificaciones:

```python
# notifications/base.py
from abc import ABC, abstractmethod

class NotificationService(ABC):
    @abstractmethod
    def notify_purchase(self, user, order) -> None: ...
    
    @abstractmethod
    def notify_course_complete(self, user, course) -> None: ...

# notifications/email_service.py
class EmailNotificationService(NotificationService):
    def notify_purchase(self, user, order) -> None:
        send_mail(...)

# notifications/webhook_service.py
class WebhookNotificationService(NotificationService):
    def notify_purchase(self, user, order) -> None:
        requests.post(webhook_url, ...)
```

### 5. Pruebas unitarias (10%)

Implementar al menos **dos pruebas unitarias** que verifiquen lógica de negocio:

```python
def test_crear_orden_sandbox_otorga_acceso_a_productos(self):
    # Crear orden → user.purchased_products debe contener los productos

def test_webhook_stripe_marca_orden_como_completada(self):
    # Simular payload de Stripe → orden.status == 'completed'
```

### 6. Calidad del código y arquitectura (15%)

- Separación clara en capas (vistas → servicios → modelos)
- Sin lógica de negocio en vistas
- Sin imports circulares entre apps
- Sin secretos en el código fuente
- Código limpio, sin archivos generados automáticamente sin contenido

### 7. Despliegue en nube + sistema de dos idiomas (15%)

- **Despliegue**: El proyecto debe estar desplegado y accesible en un servicio cloud (Railway, Render, AWS, GCP, Azure, etc.)
- **Internacionalización (i18n)**: Implementar soporte para **dos idiomas** (español + inglés)

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
