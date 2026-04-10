# Evaluación Técnica - PowerRent

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

**Arquitectura:**

Proyecto bien estructurado con `TimestampMixin`, `UUIDMixin`, y service layer completo (`EquipoService`, `ReservaService`, `PagoService`). Es uno de los proyectos con mayor madurez de servicios.

**Lo que funciona bien:**
- `TimestampMixin` y `UUIDMixin` como clases abstractas reutilizables — DRY
- `ReservaService.crear_reserva()` con `@transaction.atomic` y validaciones completas
- `EquipoService.buscar_equipos_disponibles()` con filtros de fecha, categoría y precio
- `PagoService` separando tipos de pago (alquiler vs depósito)
- `UserPassesTestMixin` para autorización en CBVs
- `select_related('categoria')` en queries de equipos
- Sistema de notificaciones con modelo `Notificacion`

**Problemas arquitectónicos:**

1. **`ICONOS` dict en el modelo `Notificacion` — mezcla presentación con dominio:**

```python
class Notificacion(models.Model):
    ICONOS = {
        'reserva_creada': 'bi-calendar-plus text-primary',  # ← clases CSS de Bootstrap!
        'pago_recibido': 'bi-credit-card text-success',
    }
```

Un modelo de datos no debe saber nada sobre clases CSS. Mover a un template tag:
```python
# templatetags/notificacion_tags.py
ICONOS = {...}

@register.filter
def icono_notificacion(tipo):
    return ICONOS.get(tipo, 'bi-bell')
```

2. **`Notificacion.Tipo.choices` duplicado en dict `ICONOS`** — si se agrega un tipo nuevo, hay que actualizar dos lugares. Viola DRY.

3. **`EquipoService.buscar_equipos_disponibles()` itera en Python en vez de filtrar en BD:**

```python
equipos_disponibles = [
    equipo.id
    for equipo in equipos     # ← list comprehension sobre QuerySet
    ...
]
```

Debe traducirse a un filtro ORM con subquery para mantener la eficiencia.

4. **Sin `select_for_update` visible en `ReservaService.crear_reserva()`** — aunque usa `@transaction.atomic`, sin `select_for_update()` sobre el equipo, dos reservas simultáneas pueden reservar el mismo equipo.


---

## Requisitos para Entrega 2 (Rúbrica)

### 1. Correcciones de la Parte 1 (10%)

- [ ] Mover `ICONOS` de modelo `Notificacion` a la capa de presentación (template tags o context processor)
- [ ] Agregar `select_for_update` en operaciones de reserva
- [ ] Eliminar duplicación entre `TextChoices` y dict `ICONOS` (DRY)

### 2. Diagrama de arquitectura actualizado (5%)

Entregar diagrama que refleje la arquitectura actual del sistema: capas (presentación, servicios, dominio, persistencia), componentes principales, y las interfaces abstractas (DIP). Formato legible (draw.io, Mermaid, PlantUML).

### 3. Servicios implementados (30%)

Ya tienen `ReservaService`. Agregar: servicio de pricing, servicio de notificaciones, servicio de reportes

Cada servicio debe:
- Estar en un archivo `services.py` dentro de su app
- Encapsular la lógica de negocio (las vistas solo delegan)
- Usar `@transaction.atomic` donde haya operaciones de escritura múltiples

### 4. Inversión de dependencias (15%)

Crear interfaz para cálculo de precios de alquiler:

```python
# pricing/base.py
from abc import ABC, abstractmethod

class PricingStrategy(ABC):
    @abstractmethod
    def calculate(self, equipo, dias) -> Decimal: ...

# pricing/standard.py
class StandardPricing(PricingStrategy):
    def calculate(self, equipo, dias) -> Decimal:
        return equipo.precio_dia * dias

# pricing/discount.py
class VolumeDiscountPricing(PricingStrategy):
    def calculate(self, equipo, dias) -> Decimal:
        base = equipo.precio_dia * dias
        if dias >= 7: return base * Decimal('0.85')  # 15% descuento
        return base
```

El `ReservaService` inyecta `PricingStrategy` según las reglas del negocio.

### 5. Pruebas unitarias (10%)

Implementar al menos **dos pruebas unitarias** que verifiquen lógica de negocio:

```python
def test_reserva_calcula_total_correctamente(self):
    # Equipo a $50/día, 3 días → total = $150

def test_no_se_puede_reservar_equipo_no_disponible(self):
    # Equipo con disponible=False → error
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
