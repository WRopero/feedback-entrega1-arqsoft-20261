# Evaluación Técnica - UNLABELED

---

## Análisis del Código Implementado

He revisado el código de su proyecto y encontré una implementación completa del sistema.

### Estructura del Proyecto

- Proyecto Django correctamente estructurado
- Docker Compose configurado para orquestación de servicios
- Modelos de datos implementados
- Vistas para lógica de negocio implementadas
- Templates HTML para interfaz de usuario
- Documentación en README presente

### Recomendaciones para la Segunda Entrega

3. **Mejorar validaciones**:
   - Agregar validadores en modelos
   - Implementar validación de permisos
   - Manejar errores apropiadamente

4. **Optimizar queries**:
   - Usar select_related y prefetch_related
   - Evitar queries N+1
   - Agregar índices en campos frecuentemente consultados

5. **Agregar tests**:
   - Tests unitarios para modelos
   - Tests de integración para vistas
   - Tests de API endpoints

---

### Análisis Detallado de Código

**Modelos:**

`Usuario` extiende `AbstractUser` con roles via `TextChoices`. `Producto` con `StockPorTalla` modela correctamente inventario por variante. El carrito tiene implementación dual: sesión (`Cart`) para anónimos y base de datos (`Carrito`) para autenticados.

**Lo que funciona bien:**
- CBVs (`ListView`, `DetailView`, `CreateView`) — uso correcto del framework
- Sistema de filtros combinados (precio, talla, búsqueda, ordenamiento) en un solo queryset
- Paginación con `paginate_by = 12`
- Carrito dual con key compuesta `producto_id:talla`
- `Q` objects para búsquedas complejas

**Problemas arquitectónicos:**

1. **Sincronización de carrito bidireccional frágil en `CustomLoginView`:**

```python
# Sesión → DB
for s_item in session_cart:
    db_cart.agregar_item(s_item['producto'], s_item['talla'], s_item['cantidad'])

# DB → Sesión
for d_item in db_cart.items.all():
    session_cart.add(d_item.producto, d_item.talla, cantidad=d_item.cantidad, override_cantidad=True)
```

Este loop bidireccional puede crear duplicados y no maneja conflictos (¿qué pasa si el mismo producto está en ambos con cantidades distintas?). Extraer a un `CartMergeService` con política clara: la DB tiene precedencia.

2. **Imports dispersos dentro de archivos:**

```python
class CustomLoginView(LoginView):
    ...
from cart.cart import Cart      # ← import en medio del archivo
from cart.models import Carrito
...
from django.contrib.auth import logout  # ← otro import suelto
```

Todos los imports deben ir al inicio del archivo (PEP8 E402).

3. **Acoplamiento entre `accounts` y `cart`** — `accounts/views.py` importa directamente de `cart`. Usar una señal `user_logged_in` o un servicio independiente.

4. **`TALLA_ORDEN` como variable global en `products/views.py`** — constante de dominio en la capa de presentación. Mover a `models.py` o `constants.py`.


---

## Requisitos para Entrega 2 (Rúbrica)

### 1. Correcciones de la Parte 1 (10%)

- [ ] Mover todos los imports al top de los archivos
- [ ] Desacoplar la sincronización de carrito (login) a un `CartMergeService`
- [ ] Mover `TALLA_ORDEN` a `models.py` o `constants.py`

### 2. Diagrama de arquitectura actualizado (5%)

Entregar diagrama que refleje la arquitectura actual del sistema: capas (presentación, servicios, dominio, persistencia), componentes principales, y las interfaces abstractas (DIP). Formato legible (draw.io, Mermaid, PlantUML).

### 3. Servicios implementados (30%)

Servicio de merge de carritos, servicio de checkout, servicio de gestión de inventario por talla

Cada servicio debe:
- Estar en un archivo `services.py` dentro de su app
- Encapsular la lógica de negocio (las vistas solo delegan)
- Usar `@transaction.atomic` donde haya operaciones de escritura múltiples

### 4. Inversión de dependencias (15%)

Crear interfaz para el carrito (sesión vs base de datos):

```python
# cart/base.py
from abc import ABC, abstractmethod

class CartBackend(ABC):
    @abstractmethod
    def add_item(self, product, size, quantity) -> None: ...
    
    @abstractmethod
    def get_items(self) -> list: ...
    
    @abstractmethod
    def clear(self) -> None: ...

# cart/session_backend.py
class SessionCartBackend(CartBackend):
    def __init__(self, session): ...

# cart/db_backend.py
class DatabaseCartBackend(CartBackend):
    def __init__(self, user): ...
```

### 5. Pruebas unitarias (10%)

Implementar al menos **dos pruebas unitarias** que verifiquen lógica de negocio:

```python
def test_agregar_producto_al_carrito_incrementa_cantidad(self):
    # Agregar mismo producto 2 veces → cantidad = 2

def test_filtro_por_talla_solo_muestra_productos_con_stock(self):
    # Filtrar por talla M → solo productos con stock > 0 en talla M
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
