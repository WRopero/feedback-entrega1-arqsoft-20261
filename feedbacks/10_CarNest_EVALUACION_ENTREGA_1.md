# Evaluación Técnica - CarNest

---

## Análisis del Código Implementado

He revisado el código de su proyecto y encontré una implementación completa del sistema.

### Estructura del Proyecto

- Proyecto Django correctamente estructurado
- Docker Compose configurado para orquestación de servicios
- Modelos de datos implementados
- Vistas para lógica de negocio implementadas
- Templates HTML para interfaz de usuario

### Recomendaciones para la Segunda Entrega

1. **Agregar README completo** con:
   - Descripción del proyecto
   - Instrucciones de instalación
   - Comandos para ejecutar con Docker
   - Ejemplos de uso

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

- [ ] Mover SECRET_KEY y DEBUG a variables de entorno
- [ ] Reemplazar `is_staff` por un sistema de roles propio
- [ ] Desacoplar app `anuncios` de `inventario` mediante capa de servicios
- [ ] Agregar transacciones atómicas en cambios de estado de vehículos
- [ ] Eliminar `usersPrueba.txt` del repositorio

### 2. Diagrama de arquitectura actualizado (5%)

Entregar diagrama que refleje la arquitectura actual del sistema: capas (presentación, servicios, dominio, persistencia), componentes principales, y las interfaces abstractas (DIP). Formato legible (draw.io, Mermaid, PlantUML).

### 3. Servicios implementados (30%)

Servicio de gestión de inventario (transiciones de estado), servicio de publicación de anuncios, servicio de ventas

Cada servicio debe:
- Estar en un archivo `services.py` dentro de su app
- Encapsular la lógica de negocio (las vistas solo delegan)
- Usar `@transaction.atomic` donde haya operaciones de escritura múltiples

### 4. Inversión de dependencias (15%)

Crear interfaz para notificaciones de estado de vehículo:

```python
# notifications/base.py
from abc import ABC, abstractmethod

class VehicleNotifier(ABC):
    @abstractmethod
    def notify_status_change(self, vehicle, old_status, new_status) -> None: ...

# notifications/email_notifier.py
class EmailVehicleNotifier(VehicleNotifier):
    def notify_status_change(self, vehicle, old_status, new_status) -> None:
        send_mail(...)

# notifications/sms_notifier.py
class SMSVehicleNotifier(VehicleNotifier):
    def notify_status_change(self, vehicle, old_status, new_status) -> None:
        # Integración con Twilio o similar
        ...
```

### 5. Pruebas unitarias (10%)

Implementar al menos **dos pruebas unitarias** que verifiquen lógica de negocio:

```python
def test_vehiculo_posteado_puede_pasar_a_en_venta(self):
    # Estado posteado → comprar_por_admin() → estado en_venta

def test_vehiculo_vendido_no_puede_volver_a_venderse(self):
    # Estado vendido → intentar vender → error
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
