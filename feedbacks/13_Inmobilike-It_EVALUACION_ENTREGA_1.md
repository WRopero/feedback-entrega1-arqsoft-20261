# Evaluación Técnica - Inmobilike-It

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

- [ ] Evaluar si `FavoriteRepository` agrega valor sobre funciones planas (Repository con solo static methods)
- [ ] Enriquecer la capa de servicio con lógica de negocio real (no solo wrapper del repository)
- [ ] Agregar autenticación en WebSocket consumer

### 2. Diagrama de arquitectura actualizado (5%)

Entregar diagrama que refleje la arquitectura actual del sistema: capas (presentación, servicios, dominio, persistencia), componentes principales, y las interfaces abstractas (DIP). Formato legible (draw.io, Mermaid, PlantUML).

### 3. Servicios implementados (30%)

Servicio de búsqueda avanzada, servicio de contacto propietario-interesado, servicio de valoraciones/comparación

Cada servicio debe:
- Estar en un archivo `services.py` dentro de su app
- Encapsular la lógica de negocio (las vistas solo delegan)
- Usar `@transaction.atomic` donde haya operaciones de escritura múltiples

### 4. Inversión de dependencias (15%)

Ya tienen Repository pattern. Formalizar con interfaz abstracta:

```python
# interactions/repositories/base.py
from abc import ABC, abstractmethod

class PropertySearchEngine(ABC):
    @abstractmethod
    def search(self, filters: dict) -> list: ...

# interactions/repositories/orm_search.py
class ORMPropertySearch(PropertySearchEngine):
    def search(self, filters: dict) -> list:
        qs = Property.objects.filter(**filters)
        return list(qs)

# interactions/repositories/elasticsearch_search.py
class ElasticPropertySearch(PropertySearchEngine):
    def search(self, filters: dict) -> list:
        # Búsqueda con Elasticsearch
        ...
```

### 5. Pruebas unitarias (10%)

Implementar al menos **dos pruebas unitarias** que verifiquen lógica de negocio:

```python
def test_agregar_favorito_duplicado_no_crea_segundo_registro(self):
    # Agregar mismo favorito 2 veces → solo 1 registro en BD

def test_usuario_no_autenticado_no_puede_enviar_inquiry(self):
    # Request sin auth → 403 o redirect a login
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
