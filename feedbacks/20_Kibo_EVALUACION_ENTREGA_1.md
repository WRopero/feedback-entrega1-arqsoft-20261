# Evaluación Técnica - Kibo E-commerce Store

> **Repositorio evaluado:** `SebastianAc02/arquitectura_sfotware_ecommerce_store`
> **Stack:** Python 3.11 · Django 4.2 · PostgreSQL 15 · Docker · DaisyUI/Tailwind CSS

---

## Análisis del Código Implementado

He revisado detalladamente el código de su proyecto y encontré una implementación sólida de un e-commerce para mascotas ("Kibo"). El proyecto demuestra comprensión real de Django MVT, aplica múltiples patrones de diseño con comentarios que los nombran explícitamente, y entrega una aplicación funcional con infraestructura Docker bien configurada.

---

### Arquitectura General

El proyecto sigue correctamente el patrón **MVT de Django** con una descomposición en tres apps funcionales:

| App | Responsabilidad |
|---|---|
| `accounts` | Autenticación e identidad del usuario |
| `store` | Catálogo, carrito, checkout, órdenes, reseñas, wishlist |
| `admin_panel` | Panel de administración con KPIs de negocio |

Esta separación vertical por dominio es apropiada para la escala del proyecto. Cada app es cohesiva y tiene sus propios `models.py`, `views.py`, `urls.py`, `forms.py` y `templates/`.

---

### Patrones de Diseño

El proyecto aplica correctamente múltiples patrones, varios de ellos nombrados explícitamente en comentarios del código, lo que evidencia comprensión real y no coincidencia:

**Snapshot Pattern** — `OrderItem.unit_price` almacena el precio al momento de la compra, preservando el histórico de órdenes aunque los precios cambien. Este es el patrón más sofisticado del proyecto y demuestra consciencia arquitectónica por encima del nivel esperado.

```python
# store/models.py
class OrderItem(models.Model):
    unit_price = models.DecimalField(...)  # Snapshot del precio al momento de compra
```

**Custom QuerySet / Manager** — `ProductQuerySet` y `ProductManager` implementan una API de consulta limpia y composable:

```python
class ProductQuerySet(models.QuerySet):
    def activos(self): ...
    def top_vendidos(self): ...          # Usa ORM aggregation con Coalesce
    def filter_by(self, category, ...): ...  # Filtros componibles
```

**Observer (Django Signals)** — `accounts/models.py` usa `post_save` para crear y sincronizar `UserProfile` automáticamente al crear un `User`. Los comentarios nombran explícitamente el patrón Observer y explican por qué evita el "God User anti-pattern".

**Finite State Machine** — `Order.status` implementa un FSM con estados definidos: `pending → confirmed → shipped → delivered → cancelled`.

**Decorator (Access Control)** — `admin_panel/views.py` implementa `@admin_required` con `functools.wraps`. Todas las vistas del panel están protegidas con este decorador en lugar de tener verificaciones duplicadas.

**Context Processor** — `store/context_processors.py` inyecta `cart_count` globalmente en todos los templates, evitando pasarlo manualmente en cada vista (principio DRY).

**Post-Redirect-Get** — Las operaciones de mutación de carrito y wishlist implementan PRG correctamente para prevenir reenvíos duplicados.

**Facade (URL Router)** — `kibo/urls.py` actúa como fachada única que delega a cada app mediante `include()`.

---

### Modelos de Datos

**Archivo:** `store/models.py`

**Fortalezas:**

- `unique_together` en `Review` y `Wishlist` para evitar duplicados
- `transaction.atomic()` en checkout para integridad transaccional
- `select_related()` y `prefetch_related()` usados correctamente (N+1 evitado)
- `Coalesce()` en agregaciones para manejo seguro de nulos
- Comando `seed_demo_data` con `get_or_create()` para idempotencia

**Mejora recomendada — índices de base de datos:**

```python
class Product(models.Model):
    class Meta:
        indexes = [
            models.Index(fields=['category', 'is_active']),
            models.Index(fields=['tipo_mascota', 'is_active']),
        ]
```

---

### Vistas y Lógica de Negocio

**Archivos:** `store/views.py`, `accounts/views.py`, `admin_panel/views.py`

**Problema arquitectónico crítico: uso exclusivo de Function-Based Views (FBVs)**

El proyecto implementa las **21 vistas** del sistema únicamente como funciones. No se usa ninguna Class-Based View (CBV) de Django, ni vistas genéricas, ni mixins.

```
store/views.py      → 11 FBVs (def home, def catalog_view, def cart_add, ...)
accounts/views.py   → 4 FBVs  (def register_view, def login_view, ...)
admin_panel/views.py → 6 FBVs (def dashboard, def product_create, ...)
```

El CRUD del panel administrativo está implementado manualmente con `if request.method == 'POST'` repetido en cada función:

```python
# admin_panel/views.py — patrón repetido en product_create, product_edit, product_delete
@admin_required
def product_create(request):
    if request.method == 'POST':
        form = ProductForm(request.POST, request.FILES)
        if form.is_valid():
            form.save()
            return redirect(...)
    else:
        form = ProductForm()
    return render(request, 'admin_panel/product_form.html', {'form': form})
```

Django provee `CreateView`, `UpdateView` y `DeleteView` exactamente para este caso. Con CBVs, el mismo CRUD sería extensible por herencia, reutilizaría lógica HTTP automáticamente, y permitiría extraer `AdminRequiredMixin` como una sola pieza de control de acceso:

```python
# Versión extensible con CBVs
from django.contrib.auth.mixins import LoginRequiredMixin
from django.views.generic import CreateView, UpdateView, DeleteView

class AdminRequiredMixin(LoginRequiredMixin):
    def dispatch(self, request, *args, **kwargs):
        if not request.user.profile.is_admin:
            return redirect('store:home')
        return super().dispatch(request, *args, **kwargs)

class ProductCreateView(AdminRequiredMixin, CreateView):
    model = Product
    form_class = ProductForm
    template_name = 'admin_panel/product_form.html'
    success_url = reverse_lazy('admin_panel:product-list')

class ProductUpdateView(AdminRequiredMixin, UpdateView):
    model = Product
    form_class = ProductForm
    template_name = 'admin_panel/product_form.html'
    success_url = reverse_lazy('admin_panel:product-list')
```

El uso de FBVs no es incorrecto en sí mismo, pero para un curso de arquitectura de software el principio de extensibilidad mediante herencia de clases es un criterio central. Las vistas funcionales no son extensibles por herencia — cada modificación requiere copiar o modificar la función directamente.

---

### Seguridad e Infraestructura

**Puntos fuertes:**

- Secretos en `.env` via `python-decouple` — sin credenciales en el código fuente
- CSRF protection en todos los formularios
- `@login_required` y `@admin_required` en todas las vistas protegidas
- `transaction.atomic()` en checkout para prevenir escrituras parciales
- Dockerfile con `python:3.11-slim` y orden de capas correcto para cache de builds
- `docker-compose.yml` con volumen nombrado para persistencia de PostgreSQL

---

### Testing

Los tres archivos `tests.py` están vacíos — solo contienen el comentario `# Create your tests here.` generado por Django. No hay ningún test unitario, de integración, ni de vistas en todo el proyecto.

Modelos como `Product.reduce_stock()`, `Cart.get_total()`, y el flujo de checkout con `transaction.atomic()` son candidatos directos para pruebas:

```python
# store/tests.py
from django.test import TestCase
from decimal import Decimal
from .models import Product

class ProductStockTestCase(TestCase):
    def test_reduce_stock_descuenta_correctamente(self):
        product = Product.objects.create(nombre="Test", precio=Decimal('50.00'), stock=10)
        product.reduce_stock(3)
        product.refresh_from_db()
        self.assertEqual(product.stock, 7)

    def test_reduce_stock_rechaza_cantidad_mayor_al_stock(self):
        product = Product.objects.create(nombre="Test", precio=Decimal('50.00'), stock=2)
        with self.assertRaises(Exception):
            product.reduce_stock(5)
```

---

### Funcionalidades Incompletas

- **Modelo `Mascota`** declarado en `accounts/models.py` pero sin vistas, formularios ni templates — el feature no es accesible para el usuario.
- **`reportlab`** está en `requirements.txt` pero la generación de PDF no está implementada en las vistas.
- **`admin.py` vacíos** en los tres apps — ningún modelo está registrado en el admin de Django.
- **Sin paginación** en el catálogo ni en la lista de órdenes.

---

## Resumen de Calificación

| Criterio | Nota |
|---|---|
| Arquitectura y Patrones de Diseño | 38/45 |
| Calidad del Código | 22/25 |
| Funcionalidad | 18/20 |
| Extensibilidad (CBVs / Herencia) | 0/10 |
| Pruebas Automatizadas | 0/10 |
| **Bonus Documentación** | +6 |
| **Total** | **78/100** |

El proyecto demuestra comprensión sólida de arquitectura Django y aplica correctamente múltiples patrones de diseño. Las deudas principales son la ausencia total de pruebas automatizadas y el uso exclusivo de Function-Based Views, que impide demostrar extensibilidad mediante herencia — un principio central en un curso de arquitectura de software.

---

## Requisitos para Entrega 2 (Rúbrica)

### 1. Correcciones de la Parte 1 (10%)

- [ ] Migrar el CRUD del `admin_panel` a `CreateView`, `UpdateView`, `DeleteView`
- [ ] Extraer `AdminRequiredMixin` como clase reutilizable
- [ ] Implementar al menos una CBV genérica en `store` (e.g., `ProductDetailView` como `DetailView`)
- [ ] Implementar el feature `Mascota` end-to-end (vista, formulario, template)
- [ ] Registrar modelos en los `admin.py` de cada app
- [ ] Agregar paginación al catálogo y lista de órdenes

### 2. Diagrama de arquitectura actualizado (5%)

Entregar diagrama que refleje la arquitectura actual: capas (presentación, lógica de negocio, dominio, persistencia), componentes principales, y las interfaces abstractas (DIP). Formato legible (draw.io, Mermaid, PlantUML).

### 3. Capa de Servicios (30%)

El `checkout_view` concentra demasiada lógica. Extraer a un `CheckoutService`:

```python
# store/services.py
class CheckoutService:
    @staticmethod
    @transaction.atomic
    def crear_orden_desde_carrito(user, cart) -> Order:
        CheckoutService._validar_stock(cart)
        order = CheckoutService._crear_orden(user, cart)
        CheckoutService._reducir_inventario(cart)
        cart.items.all().delete()
        return order
```

### 4. Inversión de Dependencias (15%)

Crear una interfaz abstracta para notificaciones o pagos:

```python
# store/services/notifications/base.py
from abc import ABC, abstractmethod

class NotificationService(ABC):
    @abstractmethod
    def notify_order_confirmed(self, order) -> None: ...

# store/services/notifications/email_service.py
class EmailNotificationService(NotificationService):
    def notify_order_confirmed(self, order) -> None:
        send_mail(subject="Orden confirmada", ...)

# store/services/notifications/mock_service.py
class MockNotificationService(NotificationService):
    def notify_order_confirmed(self, order) -> None:
        pass  # Para tests
```

El `CheckoutService` debe depender de `NotificationService` (abstracción), no de `EmailNotificationService` (concreto).

### 5. Pruebas Unitarias (10%)

Implementar al menos **dos pruebas** que verifiquen lógica de negocio:

```python
def test_checkout_descuenta_stock_correctamente(self):
    # stock=10, comprar 3 → stock debe quedar en 7

def test_checkout_falla_si_stock_insuficiente(self):
    # stock=2, intentar comprar 5 → debe lanzar excepción
```

### 6. Calidad del código y arquitectura (15%)

- Separación clara en capas (vistas → servicios → modelos)
- Sin lógica de negocio en las vistas
- CBVs para operaciones CRUD en lugar de FBVs con `if request.method == 'POST'`
- Sin secretos en el código fuente

### 7. Despliegue en nube + sistema de dos idiomas (15%)

- **Despliegue**: Proyecto accesible en un servicio cloud (Railway, Render, AWS, GCP, etc.)
- **Internacionalización (i18n)**: Soporte para español e inglés

```python
# kibo/settings.py
LANGUAGES = [
    ('es', 'Español'),
    ('en', 'English'),
]
LOCALE_PATHS = [BASE_DIR / 'locale']
USE_I18N = True

MIDDLEWARE = [
    ...
    'django.middleware.locale.LocaleMiddleware',
    ...
]
```

---

*Evaluación realizada sobre el código fuente del repositorio `SebastianAc02/arquitectura_sfotware_ecommerce_store`.*
