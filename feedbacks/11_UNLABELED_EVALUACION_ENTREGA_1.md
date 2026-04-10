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
