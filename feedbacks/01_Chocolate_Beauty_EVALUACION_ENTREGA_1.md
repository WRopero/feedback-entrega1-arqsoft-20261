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

**Mejoras sugeridas:**

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
