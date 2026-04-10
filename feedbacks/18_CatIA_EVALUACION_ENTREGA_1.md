# Evaluación Técnica - CatIA

> **Nota:** El link enviado apuntaba a una rama específica del repositorio (`/tree/main`) en lugar de la URL raíz. La evaluación automatizada no encontró el código. El repositorio correcto es `incognauta/CatIA`.

---

## Análisis del Código Implementado

CatIA es una plataforma de chat con documentos usando IA (RAG — Retrieval Augmented Generation). Es un proyecto técnicamente ambicioso y de un solo integrante. Demuestra comprensión de integración con LLMs, arquitectura de APIs y desarrollo full-stack con React.

### Arquitectura General

El backend Django está organizado en apps temáticas: `chat`, `documents`, `notebooks`, `core`. El frontend usa React/TypeScript. Tiene Docker y un `.env.example` bien documentado.

### LLM Service con Groq

La implementación del servicio de IA está correctamente aislada en `core/llm_service.py`:

```python
class GroqService:
    def generate_response(self, user_message, system_prompt=None,
                          context_documents=None, conversation_history=None):
        ...
```

El método `build_rag_context` construye el contexto a partir de documentos relevantes antes de enviar la consulta al modelo. Esto es el patrón RAG correcto: retrieve → augment → generate.

El historial de conversación se limita a los últimos 5 mensajes para evitar exceder el límite de tokens. Esta es una decisión de diseño consciente y correcta.

### Modelos de Datos

`ChatMessage` con `tokens_used` para tracking de uso es una buena práctica en sistemas con costos por token. La estructura `Notebook → ChatMessage → Document` refleja bien el modelo de dominio.

### Singleton con `get_groq_service()`

El patrón de instancia global para el cliente Groq es razonable para evitar reinicialización costosa en cada request.

---

## Áreas de Mejora

**1. Variable global mutable — problema en entornos multi-hilo**

```python
_groq_service = None

def get_groq_service():
    global _groq_service        # ← mutable global
    if _groq_service is None:
        _groq_service = GroqService()
    return _groq_service
```

En producción con Gunicorn (múltiples workers/hilos), hay una race condition en la primera inicialización. La solución más simple es usar `functools.lru_cache`:

```python
from functools import lru_cache

@lru_cache(maxsize=1)
def get_groq_service() -> GroqService:
    return GroqService()
```

**2. Constantes hardcodeadas en el módulo**

```python
GROQ_MODEL = 'llama-3.1-8b-instant'  # ← cambiar modelo requiere tocar código
MAX_TOKENS = 1024
```

Deben ser configurables desde `settings.py`:

```python
# settings.py
GROQ_MODEL = os.getenv('GROQ_MODEL', 'llama-3.1-8b-instant')
GROQ_MAX_TOKENS = int(os.getenv('GROQ_MAX_TOKENS', '1024'))
```

**3. Truncamiento de documentos a 500 caracteres sin justificación**

```python
content = doc.get('content', '')[:500]  # ← número mágico
```

500 caracteres por documento puede ser insuficiente para preguntas sobre contenido largo. Este límite debe ser un parámetro del método.

**4. Modelos duplicados en `core/models.py`**

El archivo `core/models.py` contiene `from django.db import models` seguido de `# Create your models here.` repetido tres veces. Son restos de código generado automáticamente que no se limpiaron.

**5. Solo 4 commits**

El historial de commits indica que la mayor parte del trabajo se subió en un solo commit masivo ("Initial commit"). Esto impide rastrear la evolución del proyecto, dificulta el code review y es una práctica que se debe cambiar.

---

## Recomendaciones para la Segunda Entrega

1. **Aumentar la frecuencia de commits** — cada funcionalidad nueva en su propio commit con mensaje descriptivo
2. **Hacer configurable el modelo y el max_tokens** desde variables de entorno
3. **Agregar tests** para el servicio LLM (mockeando la API de Groq) y para los endpoints de chat
4. **Implementar manejo de errores** cuando Groq no responde — retry con backoff exponencial
5. **Limpiar archivos generados automáticamente** — modelos vacíos, archivos `__init__.py` sin contenido útil
6. **Añadir límite de rate** en los endpoints de chat para evitar abuso de la API de Groq
7. **Documentar el contrato de la API** — qué campos enviar, qué se retorna, cómo manejar errores

---

## Requisitos para Entrega 2 (Rúbrica)

### 1. Correcciones de la Parte 1 (10%)

- [ ] Reemplazar singleton global mutable por `@lru_cache`
- [ ] Mover `GROQ_MODEL` y `MAX_TOKENS` a `settings.py`
- [ ] Limpiar modelos vacíos duplicados en `core/models.py`
- [ ] Aumentar frecuencia de commits

### 2. Diagrama de arquitectura actualizado (5%)

Entregar diagrama que refleje la arquitectura actual del sistema: capas (presentación, servicios, dominio, persistencia), componentes principales, y las interfaces abstractas (DIP). Formato legible (draw.io, Mermaid, PlantUML).

### 3. Servicios implementados (30%)

Servicio LLM con interfaz abstracta (DIP), servicio de procesamiento de documentos, servicio de gestión de notebooks

Cada servicio debe:
- Estar en un archivo `services.py` dentro de su app
- Encapsular la lógica de negocio (las vistas solo delegan)
- Usar `@transaction.atomic` donde haya operaciones de escritura múltiples

### 4. Inversión de dependencias (15%)

Crear interfaz abstracta para el servicio LLM:

```python
# core/llm/base.py
from abc import ABC, abstractmethod

class LLMService(ABC):
    @abstractmethod
    def generate_response(self, user_message, system_prompt=None,
                          context_documents=None) -> dict: ...

# core/llm/groq_service.py
class GroqLLMService(LLMService):
    def generate_response(self, ...) -> dict:
        # Implementación actual con Groq
        ...

# core/llm/openai_service.py
class OpenAILLMService(LLMService):
    def generate_response(self, ...) -> dict:
        # Implementación con OpenAI API
        ...
```

Agregar factory para resolución del provider según configuración.

### 5. Pruebas unitarias (10%)

Implementar al menos **dos pruebas unitarias** que verifiquen lógica de negocio:

```python
def test_build_rag_context_trunca_a_max_chars(self):
    # Documentos de 5000 chars, max_chars=3000 → contexto <= 3000

def test_generate_response_retorna_tokens_usados(self):
    # Mock de Groq API → response contiene 'tokens_used' > 0
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
