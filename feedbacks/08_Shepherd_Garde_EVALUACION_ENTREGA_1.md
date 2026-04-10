# Evaluación Técnica - Shepherd Garde

---

## Análisis del Código Implementado

He revisado el código de su plataforma de e-commerce de moda con sistema de "Drops" (lanzamientos limitados). El proyecto demuestra una arquitectura avanzada con conceptos profesionales de e-commerce.

### Modelos de Datos

**Archivos revisados:**
- `catalog/models.py` (115 líneas, 5 modelos)
- `shop/models.py` (80 líneas, 6 modelos)
- `users/models.py` (perfiles de usuario)

Su modelado de datos es **excepcional** y demuestra comprensión avanzada de sistemas de e-commerce reales.

**Clase Base TimeStampedModel - Excelente práctica:**

```python
class TimeStampedModel(models.Model):
    """Clase base abstracta con campos de auditoría."""
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    
    class Meta:
        abstract = True
```

Esto es **profesional**. Usar una clase base abstracta para auditoría evita duplicación de código.

**Modelo Collection - Sistema de Drops:**

```python
class Collection(TimeStampedModel):
    id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
    name = models.CharField(max_length=255)
    slug = models.SlugField(max_length=255, unique=True)
    release_date = models.DateTimeField(null=True, blank=True)
    end_date = models.DateTimeField(null=True, blank=True)
    
    def is_droppable(self):
        return bool(self.release_date or self.end_date)
    
    def is_active(self):
        if not self.release_date:
            return True
        return timezone.now() >= self.release_date
    
    def is_preview(self):
        if not self.release_date:
            return False
        return timezone.now() < self.release_date
```

**Aspectos destacados:**

1. **UUIDs como primary keys**: Excelente para seguridad y escalabilidad (evita enumeration attacks).

2. **Sistema de Drops flexible**: 
   - Sin `release_date` = catálogo permanente
   - Con `release_date` = lanzamiento programado (Hype)
   - Métodos de negocio claros: `is_active()`, `is_preview()`

3. **Slugs para URLs amigables**: SEO-friendly.

**Modelo ProductVariant - Sistema de SKU profesional:**

```python
class ProductVariant(TimeStampedModel):
    id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
    product = models.ForeignKey(Product, on_delete=models.CASCADE, related_name='variants')
    sku = models.CharField(max_length=100, unique=True)
    size = models.CharField(max_length=50)
    color = models.CharField(max_length=50)
    stock = models.PositiveIntegerField(default=0)
    price_override = models.DecimalField(max_digits=10, decimal_places=2, null=True, blank=True)
    
    def decrement_stock_pessimistic(self, quantity):
        """
        Bloqueo transaccional de fila para descontar inventario.
        Debe ser llamado DENTRO de un bloque transaction.atomic() junto con select_for_update().
        """
        if self.stock >= quantity:
            self.stock -= quantity
            self.save(update_fields=['stock', 'updated_at'])
            return True
        return False
```

**Esto es EXCELENTE:**

1. **SKU único por variante**: Cada combinación talla/color tiene su propio inventario.

2. **price_override**: Permite precios especiales por variante (ej: talla XL más cara).

3. **Bloqueo pesimista documentado**: El método `decrement_stock_pessimistic()` está diseñado para usarse con `select_for_update()`, evitando race conditions en compras simultáneas.

**Modelo Review - Sistema de reseñas:**

```python
class Review(TimeStampedModel):
    RATING_CHOICES = [(i, i) for i in range(1, 6)]
    
    product = models.ForeignKey(Product, on_delete=models.CASCADE, related_name='reviews')
    user = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.CASCADE, related_name='reviews')
    rating = models.PositiveSmallIntegerField(choices=RATING_CHOICES)
    title = models.CharField(max_length=120, blank=True)
    body = models.TextField(blank=True)
    
    class Meta:
        unique_together = ('product', 'user')
        ordering = ['-created_at']
```

**Fortalezas:**
- `unique_together`: Un usuario solo puede reseñar un producto una vez
- Rating de 1-5 estrellas
- Ordenamiento por fecha descendente

**Modelo Cart - Carrito de compras:**

```python
class Cart(TimeStampedModel):
    user = models.OneToOneField(settings.AUTH_USER_MODEL, on_delete=models.CASCADE, null=True, blank=True, related_name='cart')
    session_id = models.CharField(max_length=255, null=True, blank=True, unique=True)
    
    def calculate_total(self):
        return sum(item.get_subtotal() for item in self.items.all())
```

**Excelente diseño:**
- Soporta usuarios autenticados (`user`) y anónimos (`session_id`)
- `OneToOneField`: Un usuario = un carrito
- Método `calculate_total()` para obtener el total

**Modelo Order - Órdenes con estados:**

```python
class Order(TimeStampedModel):
    STATUS_CHOICES = (
        ('pending', 'Pending Payment'),
        ('paid', 'Paid / Confirmed'),
        ('shipped', 'Shipped'),
        ('cancelled', 'Cancelled / Expired'),
    )
    
    user = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.CASCADE, related_name='orders')
    ships_to = models.ForeignKey(Address, on_delete=models.PROTECT, related_name='orders_received')
    status = models.CharField(max_length=20, choices=STATUS_CHOICES, default='pending')
    total_amount = models.DecimalField(max_digits=10, decimal_places=2, default=0.00)
    payment_intent_id = models.CharField(max_length=255, null=True, blank=True)
```

**Aspectos destacados:**
- Estados claros del flujo de orden
- `PROTECT` en dirección: No se puede borrar una dirección con órdenes
- `payment_intent_id`: Integración con pasarela de pagos (Stripe)

**Modelo OrderItem - Snapshot de precios:**

```python
class OrderItem(TimeStampedModel):
    order = models.ForeignKey(Order, on_delete=models.CASCADE, related_name='items')
    variant = models.ForeignKey(ProductVariant, on_delete=models.PROTECT)
    quantity = models.PositiveIntegerField(default=1)
    price_at_purchase = models.DecimalField(max_digits=10, decimal_places=2)  # Snapshot!
```

**Esto es CRÍTICO y está bien implementado:**
- `price_at_purchase`: Guarda el precio al momento de compra
- Si el producto cambia de precio después, la orden mantiene el precio histórico
- `PROTECT`: No se puede borrar una variante que ya fue vendida

**Áreas de mejora:**

1. **Falta validación de stock en CartItem**:

```python
from django.core.exceptions import ValidationError

class CartItem(TimeStampedModel):
    # ... campos existentes ...
    
    def clean(self):
        if self.quantity > self.variant.stock:
            raise ValidationError(
                f'Solo hay {self.variant.stock} unidades disponibles de {self.variant}'
            )
    
    def save(self, *args, **kwargs):
        self.full_clean()
        super().save(*args, **kwargs)
```

2. **Agregar método para calcular rating promedio en Product**:

```python
from django.db.models import Avg

class Product(TimeStampedModel):
    # ... campos existentes ...
    
    @property
    def average_rating(self):
        avg = self.reviews.aggregate(Avg('rating'))['rating__avg']
        return round(avg, 1) if avg else 0
    
    @property
    def review_count(self):
        return self.reviews.count()
```

3. **Agregar método para proceso de checkout completo**:

```python
from django.db import transaction

class Cart(TimeStampedModel):
    @transaction.atomic
    def checkout(self, user, address):
        """
        Convierte el carrito en una orden, usando bloqueo pesimista para inventario.
        """
        # Crear orden
        order = Order.objects.create(
            user=user,
            ships_to=address,
            status='pending'
        )
        
        total = 0
        
        # Procesar cada item del carrito
        for cart_item in self.items.select_related('variant__product'):
            # Bloqueo pesimista de la variante
            variant = ProductVariant.objects.select_for_update().get(pk=cart_item.variant.pk)
            
            # Validar y reducir stock
            if not variant.decrement_stock_pessimistic(cart_item.quantity):
                raise ValueError(f'Stock insuficiente para {variant}')
            
            # Obtener precio actual
            price = variant.price_override if variant.price_override else variant.product.base_price
            
            # Crear OrderItem con snapshot de precio
            OrderItem.objects.create(
                order=order,
                variant=variant,
                quantity=cart_item.quantity,
                price_at_purchase=price
            )
            
            total += price * cart_item.quantity
        
        # Actualizar total de la orden
        order.total_amount = total
        order.save(update_fields=['total_amount'])
        
        # Limpiar carrito
        self.items.all().delete()
        
        return order
```

---

### Arquitectura del Proyecto

**Separación de apps:**
- `catalog`: Productos, colecciones, variantes, reseñas
- `shop`: Carrito, órdenes, direcciones, facturas
- `users`: Autenticación y perfiles

Esta separación es **profesional** y facilita el mantenimiento.

---

### Containerización con Docker

**Archivo:** `docker-compose.yml`

Configuración con frontend y backend separados. Excelente arquitectura.

---

### Análisis Detallado de Código

**Lo que funciona bien:**
- `TimeStampedModel` abstracto — patrón DRY aplicado correctamente
- `ProductVariant` con método `decrement_stock_pessimistic()` documentando el contrato de uso
- Documentación técnica extensa (`API_CONTRACT.md`, `ARCHITECTURE.md`, diagramas)
- `slug` en modelos para URLs semánticas
- Token refresh transparente en el cliente TypeScript con manejo de cola de requests

**Problemas arquitectónicos:**

1. **`CollectionListView.get_queryset()` retorna lista Python en vez de QuerySet — problema grave:**

```python
def get_queryset(self):
    queryset = Collection.objects.all()
    if is_preview == 'true':
        return [col for col in queryset if col.is_preview()]  # ← LISTA, no QuerySet
    return [col for col in queryset if col.is_active()]       # ← LISTA, no QuerySet
```

Esto carga TODAS las colecciones en memoria, rompe la paginación de DRF, y no permite filtros ORM adicionales. Solución:

```python
from django.utils import timezone
def get_queryset(self):
    now = timezone.now()
    if self.request.query_params.get('is_preview') == 'true':
        return Collection.objects.filter(release_date__gt=now)
    return Collection.objects.filter(
        Q(release_date__isnull=True) | Q(release_date__lte=now)
    )
```

2. **`decrement_stock_pessimistic()` en el modelo tiene contrato implícito frágil.** El método dice "debe llamarse dentro de `transaction.atomic()`" pero nada lo garantiza. Esta responsabilidad debería estar en un `CheckoutService`:

```python
class CheckoutService:
    @staticmethod
    @transaction.atomic
    def reserve_variant(variant_id, quantity):
        variant = ProductVariant.objects.select_for_update().get(id=variant_id)
        if not variant.decrement_stock_pessimistic(quantity):
            raise InsufficientStockError()
```

3. **JWT tokens en `localStorage` — vulnerable a XSS.** Si hay un ataque XSS, los tokens son accesibles. Mejor usar cookies `HttpOnly` + `SameSite=Strict`.

4. **Import de `settings` en medio de `models.py`** — aparece entre definiciones de clases. Mover al top del archivo.

5. **Sin tests.** El proyecto tiene documentación pero no tests.


---

## Requisitos para Entrega 2 (Rúbrica)

### 1. Correcciones de la Parte 1 (10%)

- [ ] Cambiar `CollectionListView.get_queryset()` para filtrar en la BD en lugar de en Python
- [ ] Mover lógica de stock pessimistic a un `CheckoutService` con transacción completa
- [ ] Capturar `IntegrityError` en creación de reviews en vez de check + save separados
- [ ] Mover import de `settings` al top de `models.py`

### 2. Diagrama de arquitectura actualizado (5%)

Entregar diagrama que refleje la arquitectura actual del sistema: capas (presentación, servicios, dominio, persistencia), componentes principales, y las interfaces abstractas (DIP). Formato legible (draw.io, Mermaid, PlantUML).

### 3. Servicios implementados (30%)

Servicio de checkout completo, servicio de inventario, servicio de notificaciones de drops

Cada servicio debe:
- Estar en un archivo `services.py` dentro de su app
- Encapsular la lógica de negocio (las vistas solo delegan)
- Usar `@transaction.atomic` donde haya operaciones de escritura múltiples

### 4. Inversión de dependencias (15%)

Crear interfaz para servicio de inventario:

```python
# inventory/base.py
from abc import ABC, abstractmethod

class InventoryService(ABC):
    @abstractmethod
    def reserve_stock(self, variant_id, quantity) -> bool: ...
    
    @abstractmethod
    def release_stock(self, variant_id, quantity) -> None: ...

# inventory/pessimistic_service.py
class PessimisticInventoryService(InventoryService):
    def reserve_stock(self, variant_id, quantity) -> bool:
        with transaction.atomic():
            variant = ProductVariant.objects.select_for_update().get(id=variant_id)
            ...

# inventory/optimistic_service.py
class OptimisticInventoryService(InventoryService):
    def reserve_stock(self, variant_id, quantity) -> bool:
        # Usar versioning/CAS para control optimista
        ...
```

### 5. Pruebas unitarias (10%)

Implementar al menos **dos pruebas unitarias** que verifiquen lógica de negocio:

```python
def test_decrement_stock_rechaza_si_insuficiente(self):
    # Variante con stock=2, intentar reservar 5 → retorna False

def test_collection_preview_no_permite_compra(self):
    # Colección con release_date futuro → no se puede hacer checkout
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
