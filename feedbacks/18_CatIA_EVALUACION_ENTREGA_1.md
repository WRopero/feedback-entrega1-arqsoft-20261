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
