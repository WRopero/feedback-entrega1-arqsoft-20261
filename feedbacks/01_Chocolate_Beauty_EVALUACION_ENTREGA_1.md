# Evaluación Técnica - Chocolate Beauty

---

## Análisis del Código Implementado

He revisado detalladamente el código de su proyecto y encontré una implementación muy sólida de un e-commerce de productos de skincare. El proyecto demuestra comprensión profunda de Django REST Framework y arquitectura de aplicaciones modernas.

### Modelos de Datos

**Archivo:** `backend/ecommerce/models.py` (286 líneas, 11 modelos)

Su modelado de datos es uno de los más completos de la clase. Implementaron:

- **Producto**: Con validadores de precio (`MinValueValidator`), método `esta_disponible()`, y property `calificacion_promedio` que calcula dinámicamente el promedio de reseñas.
- **Carrito e ItemCarrito**: Separación correcta entre el carrito y sus items, con método `precio_total` como property.
- **Pedido y DetallePedido**: Uso de `TextChoices` para estados, snapshot del precio en `DetallePedido` (excelente práctica para mantener histórico), y métodos de negocio como `confirmar()` y `cancelar()`.
- **Envio**: Modelo separado con su propio ciclo de vida de estados.
- **PerfilUsuario**: Manejo de roles (cliente, tienda, admin) con properties `es_tienda()` y `es_admin()`.
- **Reseña**: Con constraint `unique_together` para evitar múltiples reseñas del mismo usuario al mismo producto.

**Fortalezas identificadas:**

```python
# Excelente uso de properties para lógica calculada
@property
def calificacion_promedio(self) -> float:
    reseñas = self.reseñas.all()
    if not reseñas.exists():
        return 0.0
    total = sum(r.calificacion for r in reseñas)
    return round(total / reseñas.count(), 1)
```

El uso de `update_fields` en los métodos `save()` es una optimización importante:

```python
def actualizar_stock(self, nuevo_stock: int) -> None:
    if nuevo_stock < 0:
        raise ValueError('El stock no puede ser negativo')
    self.stock = nuevo_stock
    self.save(update_fields=['stock'])  # Solo actualiza este campo
```

**Áreas de mejora:**

1. En el modelo `Reseña`, falta validación del rango de calificación en el modelo mismo:

```python
# Mejora sugerida
class Reseña(models.Model):
    calificacion = models.PositiveSmallIntegerField(
        validators=[
            MinValueValidator(1),
            MaxValueValidator(5)  # Agregar límite superior
        ],
    )
```

2. Considerar agregar índices para optimizar consultas frecuentes:

```python
class Producto(models.Model):
    # ... campos existentes ...
    
    class Meta:
        indexes = [
            models.Index(fields=['categoria', 'activo']),  # Para filtros
            models.Index(fields=['-created_at']),  # Si agregan timestamp
        ]
```

---

### Vistas y Lógica de Negocio

**Archivo:** `backend/ecommerce/views.py` (607 líneas, 21 funciones/métodos)

La implementación de las vistas es profesional y demuestra conocimiento avanzado de Django REST Framework.

**Aspectos destacados:**

1. **Permisos personalizados**: Implementaron `IsTienda` y `IsOwnerOrAdmin`, lo cual es fundamental para seguridad.

```python
class IsTienda(permissions.BasePermission):
    def has_permission(self, request, view) -> bool:
        if not request.user.is_authenticated:
            return False
        try:
            return request.user.perfil.rol in (
                PerfilUsuario.Rol.TIENDA,
                PerfilUsuario.Rol.ADMIN,
            )
        except PerfilUsuario.DoesNotExist:
            return False
```

2. **Transacciones atómicas**: Uso correcto de `@transaction.atomic` en `CheckoutPedidoView` para garantizar consistencia.

3. **Generación de PDF**: Implementaron generación de facturas con ReportLab directamente en el backend.

4. **Optimización de queries**: Uso de `select_related` y `prefetch_related` para evitar N+1:

```python
def get_queryset(self):
    return (
        Pedido.objects
        .filter(detalles__producto__tienda=user)
        .select_related('envio', 'usuario')
        .prefetch_related('detalles')
        .distinct()
        .order_by('-fecha')
    )
```

5. **Validación robusta en checkout**: Verifican stock, productos activos, y manejan errores de inventario antes de crear el pedido.

**Problemas arquitectónicos críticos:**

1. **`views.py` es un God File (607 líneas) — viola SRP.** Auth, productos, pedidos, envíos y reseñas viven en un solo archivo. Esto hace el código difícil de mantener, testear y navegar. Debe separarse en módulos:

```
ecommerce/views/
    __init__.py
    auth.py          # CustomTokenObtainPairView, RegistroClienteView, etc.
    products.py      # ProductoListCreateView, ProductoDetailView, etc.
    orders.py        # CheckoutPedidoView, MisPedidosListView, etc.
    reviews.py       # ReseñaListCreateView
```

2. **No hay capa de servicios — lógica de negocio en las vistas.** `CheckoutPedidoView.post()` tiene ~100 líneas orquestando validación, descuento de stock, creación de pedido, generación de PDF y actualización de perfil. Esto debe extraerse a un `CheckoutService`:

```python
# services/checkout_service.py
class CheckoutService:
    @staticmethod
    @transaction.atomic
    def process_checkout(user, items, envio_data, perfil_data) -> Pedido:
        validated_items = CheckoutService._validate_items(items)
        pedido = CheckoutService._create_order(user, validated_items)
        CheckoutService._create_shipping(pedido, envio_data)
        CheckoutService._generate_invoice(pedido)
        return pedido
```

3. **`_build_invoice_pdf()` en `views.py`** — una función de 40 líneas de generación de PDF con ReportLab no tiene nada que ver con HTTP. Debe estar en `services/invoice_service.py`.

4. **`calificacion_promedio` hace N+1 implícito** — itera todas las reseñas en Python:

```python
# Actual (carga todos los objetos en memoria)
total = sum(r.calificacion for r in reseñas)

# Correcto (delega al motor SQL)
from django.db.models import Avg
return self.reseñas.aggregate(avg=Avg('calificacion'))['avg'] or 0.0
```

5. **`categoria` como `CharField` libre** — permite `"Skincare"`, `"skincare"`, `"SKINCARE"` como categorías distintas. Debe ser `ForeignKey` a un modelo `Categoria` con `unique=True` en el nombre.

6. **Import dentro de función en `ReseñaListCreateView.perform_create`:**
```python
# Actual (malo)
from rest_framework.exceptions import PermissionDenied  # import lazy innecesario
# Correcto: mover al top del archivo
```

7. **Archivos media en el repositorio** — 20+ imágenes en `backend/media/products/`. Las imágenes no deben estar en git. Agregar `media/` al `.gitignore`.

**Mejoras de optimización:**

1. En `CheckoutPedidoView`, el manejo de stock podría mejorarse usando `F()` expressions para evitar race conditions:

```python
from django.db.models import F

# En lugar de:
producto.stock -= cantidad
producto.save(update_fields=['stock'])

# Mejor:
Producto.objects.filter(id=producto.id).update(
    stock=F('stock') - cantidad
)
# Luego verificar que no quedó negativo
producto.refresh_from_db()
if producto.stock < 0:
    raise ValidationError("Stock insuficiente")
```

2. Agregar paginación a las listas de productos:

```python
from rest_framework.pagination import PageNumberPagination

class ProductoPagination(PageNumberPagination):
    page_size = 20
    page_size_query_param = 'page_size'
    max_page_size = 100

class ProductoListCreateView(generics.ListCreateAPIView):
    pagination_class = ProductoPagination
    # ... resto del código
```

---

### Serializers

**Archivo:** `backend/ecommerce/serializers.py` (337 líneas)

Los serializers están muy bien estructurados con validaciones apropiadas.

**Puntos fuertes:**

1. Validación de contraseñas coincidentes en registro
2. Uso de `SerializerMethodField` para datos calculados
3. Transacciones atómicas en creación de usuarios
4. Validación de email único

**Mejora recomendada:**

Agregar validación de fortaleza de contraseña:

```python
import re

class RegistroClienteSerializer(serializers.ModelSerializer):
    def validate_password(self, value):
        if len(value) < 8:
            raise serializers.ValidationError(
                "La contraseña debe tener al menos 8 caracteres"
            )
        if not re.search(r'[A-Z]', value):
            raise serializers.ValidationError(
                "La contraseña debe contener al menos una mayúscula"
            )
        if not re.search(r'[0-9]', value):
            raise serializers.ValidationError(
                "La contraseña debe contener al menos un número"
            )
        return value
```

---

### Containerización con Docker

**Archivo:** `docker-compose.yml`

Excelente configuración de Docker con:

- Servicio PostgreSQL con healthcheck
- Volúmenes para persistencia de datos y media files
- `depends_on` con `condition: service_healthy`
- Separación de frontend y backend en contenedores distintos
- Variables de entorno correctamente configuradas

**Observación importante:**

En el README mencionan cambiar `DJANGO_DEBUG=True` para desarrollo, pero esto debería manejarse automáticamente con el archivo `.env`. Asegúrense de que `.env.example` tenga valores apropiados para desarrollo.

---

### Testing

Encontré que implementaron tests (`tests.py` con 9025 bytes), lo cual es excelente y va más allá de lo requerido. Esto demuestra madurez en el desarrollo de software.

**Recomendación para expandir tests:**

```python
from django.test import TestCase
from decimal import Decimal
from .models import Producto, Pedido, DetallePedido

class PedidoTestCase(TestCase):
    def test_calcular_total_pedido(self):
        """Verificar que el total se calcula correctamente"""
        pedido = Pedido.objects.create(usuario=self.user)
        producto = Producto.objects.create(
            nombre="Test", 
            precio=Decimal('100.00'),
            stock=10
        )
        DetallePedido.objects.create(
            pedido=pedido,
            producto=producto,
            cantidad=2,
            precio_unitario=producto.precio
        )
        
        total = pedido.calcular_total()
        self.assertEqual(total, Decimal('200.00'))
    
    def test_stock_se_reduce_en_checkout(self):
        """Verificar que el stock se reduce al confirmar pedido"""
        producto = Producto.objects.create(
            nombre="Test",
            precio=Decimal('50.00'),
            stock=10
        )
        # Simular checkout con 3 unidades
        # ... código de checkout ...
        producto.refresh_from_db()
        self.assertEqual(producto.stock, 7)
```

---

## Requisitos para Entrega 2 (Rúbrica)

### 1. Correcciones de la Parte 1 (10%)

- [ ] Separar `views.py` en módulos por dominio (auth, products, orders, reviews)
- [ ] Extraer lógica de checkout a un `CheckoutService`
- [ ] Mover `_build_invoice_pdf()` a una capa de servicios
- [ ] Reemplazar `calificacion_promedio` con `Avg()` de Django ORM
- [ ] Convertir `categoria` de `CharField` a `ForeignKey` a un modelo `Categoria`
- [ ] Quitar imports dentro de funciones (mover al top del archivo)
- [ ] Quitar archivos media del repositorio git

### 2. Diagrama de arquitectura actualizado (5%)

Entregar diagrama que refleje la arquitectura actual del sistema: capas (presentación, servicios, dominio, persistencia), componentes principales, y las interfaces abstractas (DIP). Formato legible (draw.io, Mermaid, PlantUML).

### 3. Servicios implementados (30%)

Servicio de checkout, servicio de generación de facturas PDF, servicio de notificaciones (email al comprar)

Cada servicio debe:
- Estar en un archivo `services.py` dentro de su app
- Encapsular la lógica de negocio (las vistas solo delegan)
- Usar `@transaction.atomic` donde haya operaciones de escritura múltiples

### 4. Inversión de dependencias (15%)

Crear una interfaz abstracta para el servicio de pagos, con dos implementaciones concretas:

```python
# services/payment/base.py
from abc import ABC, abstractmethod

class PaymentGateway(ABC):
    @abstractmethod
    def process_payment(self, amount, currency, token) -> dict: ...
    
    @abstractmethod
    def refund(self, transaction_id) -> dict: ...

# services/payment/stripe_gateway.py
class StripeGateway(PaymentGateway):
    def process_payment(self, amount, currency, token) -> dict:
        # Implementación con Stripe API
        ...

# services/payment/mock_gateway.py
class MockGateway(PaymentGateway):
    def process_payment(self, amount, currency, token) -> dict:
        return {"status": "approved", "transaction_id": "MOCK-001"}
```

El `CheckoutService` debe depender de `PaymentGateway` (abstracción), no de `StripeGateway` (concreto).

### 5. Pruebas unitarias (10%)

Implementar al menos **dos pruebas unitarias** que verifiquen lógica de negocio:

```python
def test_checkout_descuenta_stock_correctamente(self):
    # Producto con stock=10, comprar 3 → stock debe ser 7

def test_checkout_rechaza_si_stock_insuficiente(self):
    # Producto con stock=2, intentar comprar 5 → debe fallar
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
