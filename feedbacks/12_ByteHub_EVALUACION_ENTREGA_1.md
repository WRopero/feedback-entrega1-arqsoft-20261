# Evaluación Técnica - ByteHub

---

## Análisis del Código Implementado

He revisado el código de su plataforma de e-commerce de tecnología. El proyecto demuestra una arquitectura **profesional** con código de nivel empresarial, custom managers, optimizaciones de queries y validaciones robustas.

### Modelos de Datos

**Archivos revisados:**
- `store/models.py` (386 líneas, 8 modelos con managers personalizados)
- `ByteHub/orders/models.py` (153 líneas, 2 modelos)
- `accounts/models.py` (perfiles de usuario)
- `pages/models.py` (categorías y páginas)

Su código es **excepcional** y demuestra conocimiento profesional de Django.

**Custom Managers - Nivel Profesional:**

```python
class ProductManager(models.Manager):
    """Custom manager with reusable product queries."""
    
    def get_active_products(self):
        return (
            self.filter(is_available=True)
            .select_related('category')
        )
    
    def search_active_products_by_name(self, query):
        cleaned_query = query.strip()
        if not cleaned_query:
            return self.get_active_products()
        return self.get_active_products().filter(name__icontains=cleaned_query)
    
    def get_public_detail(self):
        """Return optimized queryset for public product detail pages."""
        return (
            self.filter(is_available=True)
            .select_related('category')
            .prefetch_related(
                Prefetch(
                    'reviews',
                    queryset=Review.objects.filter(
                        is_verified_purchase=True,
                    ).select_related('user'),
                    to_attr='verified_reviews',
                )
            )
            .annotate(
                verified_avg_rating=Avg(
                    'reviews__rating',
                    filter=Q(reviews__is_verified_purchase=True),
                )
            )
        )
```

**Esto es EXCELENTE:**

1. **Custom managers**: Encapsulan lógica de queries reutilizables
2. **Prefetch con filtro**: `Prefetch()` con queryset personalizado evita N+1 queries
3. **to_attr**: Cachea reviews verificadas en `verified_reviews`
4. **Annotate con filtro**: Calcula rating promedio solo de compras verificadas
5. **select_related**: Optimiza joins con categoría

**Modelo Product - Código Profesional:**

```python
class Product(models.Model):
    name = models.CharField(_('name'), max_length=200)
    slug = models.SlugField(_('slug'), max_length=220, unique=True)
    description = models.TextField(_('description'), blank=True)
    brand = models.CharField(_('brand'), max_length=100, blank=True)
    price = models.DecimalField(
        _('price'), max_digits=10, decimal_places=2,
        validators=[MinValueValidator(Decimal('0.01'))],
    )
    stock = models.PositiveIntegerField(_('stock'), default=0)
    category = models.ForeignKey(
        Category,
        on_delete=models.PROTECT,
        related_name='products',
    )
    created_by = models.ForeignKey(
        settings.AUTH_USER_MODEL,
        on_delete=models.PROTECT,
        related_name='managed_products',
    )
    
    objects = ProductManager()
    
    def clean(self):
        super().clean()
        if self.price is not None and self.price <= 0:
            raise ValidationError(
                {'price': _("Price must be greater than zero.")}
            )
    
    def avg_rating(self):
        """Return average rating from verified reviews.
        
        Uses the pre-annotated verified_avg_rating value when the
        object was fetched via get_public_detail() to avoid an
        extra aggregate query.
        """
        if hasattr(self, 'verified_avg_rating'):
            value = self.verified_avg_rating
        else:
            result = self.reviews.filter(
                is_verified_purchase=True,
            ).aggregate(average=Avg('rating'))
            value = result['average']
        return float(value) if value is not None else 0.0
```

**Aspectos destacados:**

1. **Internacionalización**: Uso de `gettext_lazy` para i18n
2. **Validators en campo**: `MinValueValidator` en precio
3. **Validación en clean()**: Doble validación de precio positivo
4. **avg_rating() optimizado**: Reutiliza anotación si existe, evita query extra
5. **PROTECT en relaciones**: No se pueden borrar categorías/usuarios con productos
6. **Custom manager asignado**: `objects = ProductManager()`

**Modelo CartItem - Validaciones Robustas:**

```python
class CartItem(models.Model):
    cart = models.ForeignKey(Cart, on_delete=models.CASCADE, related_name='items')
    product = models.ForeignKey(Product, on_delete=models.CASCADE, related_name='cart_items')
    quantity = models.PositiveIntegerField(_('quantity'), default=1)
    
    class Meta:
        constraints = [
            models.UniqueConstraint(
                fields=['cart', 'product'],
                name='store_cart_item_unique_cart_product',
            )
        ]
    
    def clean(self):
        super().clean()
        if self.quantity > self.product.stock:
            raise ValidationError(
                {
                    'quantity': _(
                        'Quantity cannot exceed available stock.',
                    )
                }
            )
```

**Excelente:**
- **UniqueConstraint**: Un producto solo puede estar una vez por carrito
- **Validación de stock**: Previene agregar más cantidad que la disponible
- **Mensajes traducibles**: Preparado para múltiples idiomas

**Modelo Order - Validación de Totales:**

```python
class Order(models.Model):
    STATUS_PENDING = 'pending'
    STATUS_CONFIRMED = 'confirmed'
    STATUS_SHIPPED = 'shipped'
    STATUS_DELIVERED = 'delivered'
    STATUS_CANCELLED = 'cancelled'
    
    ORDER_STATUS_CHOICES = [
        (STATUS_PENDING, _('Pending')),
        (STATUS_CONFIRMED, _('Confirmed')),
        (STATUS_SHIPPED, _('Shipped')),
        (STATUS_DELIVERED, _('Delivered')),
        (STATUS_CANCELLED, _('Cancelled')),
    ]
    
    subtotal = models.DecimalField(
        _('subtotal'),
        max_digits=10,
        decimal_places=2,
        default=Decimal('0.00'),
        validators=[MinValueValidator(Decimal('0.00'))],
    )
    shipping_cost = models.DecimalField(
        _('shipping cost'),
        max_digits=10,
        decimal_places=2,
        default=Decimal('0.00'),
        validators=[MinValueValidator(Decimal('0.00'))],
    )
    total_amount = models.DecimalField(
        _('total amount'),
        max_digits=10,
        decimal_places=2,
        default=Decimal('0.00'),
        validators=[MinValueValidator(Decimal('0.00'))],
    )
    
    def clean(self):
        super().clean()
        expected_total = (self.subtotal or Decimal('0.00')) + (
            self.shipping_cost or Decimal('0.00')
        )
        if (
            self.total_amount is not None
            and self.total_amount != expected_total
        ):
            raise ValidationError(
                {
                    'total_amount': _(
                        'Total amount must equal subtotal plus shipping cost.'
                    )
                }
            )
```

**Esto es CRÍTICO y está perfectamente implementado:**
- Validación de integridad: `total = subtotal + shipping`
- Previene manipulación de totales desde el frontend
- Estados completos del flujo de orden

**Modelo Review - Sistema de Reseñas Verificadas:**

```python
class Review(models.Model):
    product = models.ForeignKey(Product, on_delete=models.CASCADE, related_name='reviews')
    user = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.CASCADE, related_name='reviews')
    rating = models.PositiveSmallIntegerField(
        _('rating'),
        validators=[
            MinValueValidator(1, message=_("Rating must be between 1 and 5.")),
            MaxValueValidator(5, message=_("Rating must be between 1 and 5.")),
        ],
    )
    title = models.CharField(_('title'), max_length=120, blank=True)
    body = models.TextField(_('review body'), blank=True)
    is_verified_purchase = models.BooleanField(_('verified purchase'), default=False)
    
    class Meta:
        constraints = [
            models.UniqueConstraint(
                fields=['product', 'user'],
                name='store_review_unique_product_user',
            )
        ]
```

**Excelente diseño:**
- `is_verified_purchase`: Distingue reseñas de compradores reales
- Validadores con mensajes personalizados
- UniqueConstraint: Un usuario solo puede reseñar un producto una vez

**Custom Managers para Órdenes:**

```python
class OrderManager(models.Manager):
    def get_user_orders_with_details(self, user_id):
        """Return orders for a user with items and products prefetched."""
        return (
            self.filter(user_id=user_id)
            .prefetch_related(
                Prefetch(
                    'items',
                    queryset=OrderItem.objects.select_related('product'),
                )
            )
            .order_by('-created_at')
        )
```

**Profesional**: Prefetch anidado para evitar N+1 queries en items → products.

**Áreas de mejora (menores):**

1. **Agregar método para verificar compra automáticamente**:

```python
class Review(models.Model):
    # ... campos existentes ...
    
    def save(self, *args, **kwargs):
        # Auto-verificar si el usuario compró el producto
        if not self.pk:  # Solo en creación
            has_purchased = OrderItem.objects.filter(
                order__user=self.user,
                order__status__in=['confirmed', 'shipped', 'delivered'],
                product=self.product
            ).exists()
            self.is_verified_purchase = has_purchased
        super().save(*args, **kwargs)
```

2. **Agregar método para proceso de checkout con transacciones**:

```python
from django.db import transaction

class Cart(models.Model):
    @transaction.atomic
    def checkout(self, shipping_address):
        """Convert cart to order with stock validation."""
        if not self.items.exists():
            raise ValueError("Cart is empty")
        
        # Crear orden
        order = Order.objects.create(
            user=self.user,
            shipping_address=shipping_address,
            status='pending'
        )
        
        subtotal = Decimal('0.00')
        
        # Procesar items con bloqueo
        for cart_item in self.items.select_related('product'):
            product = Product.objects.select_for_update().get(pk=cart_item.product.pk)
            
            # Validar stock
            if product.stock < cart_item.quantity:
                raise ValueError(f'Insufficient stock for {product.name}')
            
            # Reducir stock
            product.stock -= cart_item.quantity
            product.save(update_fields=['stock'])
            
            # Crear OrderItem
            OrderItem.objects.create(
                order=order,
                product=product,
                quantity=cart_item.quantity,
                unit_price=product.price
            )
            
            subtotal += product.price * cart_item.quantity
        
        # Actualizar totales
        order.subtotal = subtotal
        order.shipping_cost = Decimal('5.00')  # O calcular dinámicamente
        order.total_amount = subtotal + order.shipping_cost
        order.save()
        
        # Limpiar carrito
        self.items.all().delete()
        
        return order
```

---

### Arquitectura del Proyecto

**Separación de apps:**
- `store`: Productos, carrito, reseñas
- `orders`: Órdenes y order items (separado de store)
- `accounts`: Autenticación y perfiles
- `pages`: Categorías y contenido estático

Esta separación es **profesional** y sigue principios SOLID.

---

### Containerización con Docker

**Archivo:** `docker-compose.yml`

Configuración correcta con PostgreSQL y templates.

---

## Requisitos para Entrega 2 (Rúbrica)

### 1. Correcciones de la Parte 1 (10%)

- [ ] Unificar la estructura de carpetas (accounts fuera del proyecto Django)
- [ ] Agregar capa de servicios para lógica de órdenes

### 2. Diagrama de arquitectura actualizado (5%)

Entregar diagrama que refleje la arquitectura actual del sistema: capas (presentación, servicios, dominio, persistencia), componentes principales, y las interfaces abstractas (DIP). Formato legible (draw.io, Mermaid, PlantUML).

### 3. Servicios implementados (30%)

Servicio de órdenes (crear, confirmar, cancelar), servicio de email, servicio de catálogo con búsqueda

Cada servicio debe:
- Estar en un archivo `services.py` dentro de su app
- Encapsular la lógica de negocio (las vistas solo delegan)
- Usar `@transaction.atomic` donde haya operaciones de escritura múltiples

### 4. Inversión de dependencias (15%)

Crear interfaz para envío de emails (password reset, notificaciones):

```python
# core/email/base.py
from abc import ABC, abstractmethod

class EmailBackend(ABC):
    @abstractmethod
    def send(self, to, subject, body, html=None) -> bool: ...

# core/email/smtp_backend.py
class SMTPEmailBackend(EmailBackend):
    def send(self, to, subject, body, html=None) -> bool:
        send_mail(subject, body, settings.DEFAULT_FROM_EMAIL, [to])

# core/email/sendgrid_backend.py
class SendGridEmailBackend(EmailBackend):
    def send(self, to, subject, body, html=None) -> bool:
        # Integración con SendGrid API
        ...
```

### 5. Pruebas unitarias (10%)

Implementar al menos **dos pruebas unitarias** que verifiquen lógica de negocio:

```python
def test_crear_usuario_con_email_duplicado_case_insensitive_falla(self):
    # Crear user@test.com → crear User@Test.COM → IntegrityError

def test_orden_lista_solo_ordenes_del_usuario_autenticado(self):
    # User A tiene 2 órdenes, User B tiene 1 → User A solo ve 2
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
