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
