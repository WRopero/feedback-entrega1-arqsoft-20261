# Evaluación Técnica - DocsMind

---

## Análisis del Código Implementado

He revisado el código de su plataforma de chat con documentos usando IA. El proyecto demuestra una arquitectura bien diseñada para procesamiento de documentos con OCR/conversión a Markdown y sistema de suscripciones.

### Modelos de Datos

**Archivos revisados:**
- `backend/documents/models.py` (97 líneas, 1 modelo)
- `backend/chats/models.py` (61 líneas, 2 modelos)
- `backend/subscription/models.py` (136 líneas, 2 modelos)
- `backend/users/models.py` (perfiles de usuario)

Su modelado de datos es limpio y bien pensado para un sistema de procesamiento de documentos con IA.

**Modelo Document - Procesamiento Asíncrono:**

```python
class Document(models.Model):
    class FileType(models.TextChoices):
        PDF = "pdf", "PDF"
        IMAGE = "image", "Imagen"
        DOCX = "docx", "Word (.docx)"
        TEXT = "text", "Texto plano"
        
        @classmethod
        def from_extension(cls, filename: str) -> str:
            import os
            ext = os.path.splitext(filename)[1].lower()
            file_type = Document.EXTENSION_MAP.get(ext)
            if file_type is None:
                raise ValueError(f"Tipo de archivo no soportado: {ext}")
            return file_type
    
    class Status(models.TextChoices):
        PENDING = "pending", "Pendiente"
        PROCESSING = "processing", "Procesando"
        COMPLETED = "completed", "Completado"
        FAILED = "failed", "Fallido"
    
    id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
    user = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.CASCADE, related_name="documents")
    file = models.FileField(upload_to=document_upload_path)
    file_type = models.CharField(max_length=10, choices=FileType.choices)
    file_size = models.PositiveBigIntegerField(help_text="Tamaño en bytes")
    markdown_content = models.TextField(blank=True, default="")
    status = models.CharField(max_length=20, choices=Status.choices, default=Status.PENDING)
    error_message = models.TextField(blank=True, default="")
```

**Aspectos destacados:**

1. **UUIDs como primary keys**: Excelente para seguridad.

2. **Estados de procesamiento**: Sistema asíncrono bien diseñado (PENDING → PROCESSING → COMPLETED/FAILED).

3. **Método classmethod para detección de tipo**:
```python
@classmethod
def detect_file_type(cls, filename: str) -> str:
    import os
    ext = os.path.splitext(filename)[1].lower()
    file_type = cls.EXTENSION_MAP.get(ext)
    if file_type is None:
        raise ValueError(f"Tipo de archivo no soportado: {ext}")
    return file_type
```

4. **EXTENSION_MAP**: Mapeo claro de extensiones a tipos de archivo.

5. **markdown_content**: Almacena el contenido procesado para consultas de IA.

6. **Custom upload path**:
```python
def document_upload_path(instance, filename):
    """Organiza archivos por usuario: documents/<user_id>/<filename>"""
    return f"documents/{instance.user_id}/{filename}"
```

**Modelo Chat - Conversaciones con Documentos:**

```python
class Chat(models.Model):
    id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
    user = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.CASCADE, related_name="chats")
    document = models.ForeignKey(Document, on_delete=models.CASCADE, related_name="chats")
    title = models.CharField(max_length=255)
    created_at = models.DateTimeField(auto_now_add=True)
```

**Diseño inteligente:**
- ForeignKey (no OneToOne) permite múltiples chats por documento
- Relación usuario-documento-chat bien estructurada

**Modelo Message - Historial de Conversación:**

```python
class Message(models.Model):
    class Role(models.TextChoices):
        USER = "user", "Usuario"
        ASSISTANT = "assistant", "Asistente"
    
    id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
    chat = models.ForeignKey(Chat, on_delete=models.CASCADE, related_name="messages")
    role = models.CharField(max_length=10, choices=Role.choices)
    content = models.TextField()
    created_at = models.DateTimeField(auto_now_add=True)
    
    class Meta:
        ordering = ["created_at"]
```

**Excelente para integración con IA:**
- Role USER/ASSISTANT compatible con APIs de OpenAI/Anthropic
- Ordenamiento cronológico para mantener contexto

**Sistema de Suscripciones - Modelo Plan:**

```python
class Plan(models.Model):
    class PlanType(models.TextChoices):
        FREE = "free", "Free"
        PRO = "pro", "Pro"
    
    PLAN_CATALOG = {
        PlanType.FREE: {
            "name": "Free",
            "price": "0.00",
            "max_documents": 5,
            "max_storage_mb": 10,
        },
        PlanType.PRO: {
            "name": "Pro",
            "price": "9.99",
            "max_documents": 100,
            "max_storage_mb": 1024,
        },
    }
    
    plan_type = models.CharField(max_length=20, choices=PlanType.choices, unique=True)
    max_documents = models.PositiveIntegerField(null=True, blank=True)
    max_storage_mb = models.PositiveIntegerField(null=True, blank=True)
```

**Profesional:**
- PLAN_CATALOG como constante de clase
- Límites configurables por plan

**Modelo Subscription - Métodos de Negocio:**

```python
class Subscription(models.Model):
    user = models.OneToOneField(settings.AUTH_USER_MODEL, on_delete=models.CASCADE, related_name="subscription")
    plan = models.ForeignKey(Plan, on_delete=models.PROTECT, related_name="subscriptions")
    status = models.CharField(max_length=20, choices=Status.choices, default=Status.ACTIVE)
    
    def save(self, *args, **kwargs):
        """Asigna el plan Free por defecto si no se especifica ningún plan."""
        if self.plan is None:
            self.plan = self._get_free_plan()
        super().save(*args, **kwargs)
    
    def upgrade(self, plan: "Plan") -> None:
        """Sube al usuario al plan indicado y reactiva la suscripción."""
        self.plan = plan
        self.status = Subscription.Status.ACTIVE
        self.expires_at = None
        self.save(update_fields=["plan", "status", "expires_at", "updated_at"])
    
    def cancel(self) -> None:
        """Cancela la suscripción y hace downgrade al plan Free."""
        free_plan = self._get_free_plan()
        Subscription.objects.filter(pk=self.pk).update(
            status=Subscription.Status.CANCELLED,
            plan=free_plan,
        )
```

**Excelente diseño:**
- OneToOneField: Un usuario = una suscripción
- Plan Free por defecto en save()
- Métodos upgrade() y cancel() bien implementados
- Uso de update() en cancel() para evitar disparar signals

**Áreas de mejora:**

1. **Agregar validación de límites de plan**:

```python
class Document(models.Model):
    def save(self, *args, **kwargs):
        if not self.pk:  # Solo en creación
            # Validar límite de documentos
            user_docs_count = Document.objects.filter(user=self.user).count()
            max_docs = self.user.subscription.plan.max_documents
            
            if user_docs_count >= max_docs:
                raise ValidationError(
                    f'Has alcanzado el límite de {max_docs} documentos de tu plan'
                )
            
            # Validar límite de almacenamiento
            total_storage = Document.objects.filter(
                user=self.user
            ).aggregate(total=Sum('file_size'))['total'] or 0
            
            max_storage_bytes = self.user.subscription.plan.max_storage_mb * 1024 * 1024
            
            if total_storage + self.file_size > max_storage_bytes:
                raise ValidationError(
                    f'Has excedido el límite de almacenamiento de tu plan'
                )
        
        super().save(*args, **kwargs)
```

2. **Agregar método para procesar documento**:

```python
class Document(models.Model):
    def process(self):
        """Inicia el procesamiento del documento."""
        self.status = self.Status.PROCESSING
        self.save(update_fields=['status'])
        
        try:
            # Aquí iría la lógica de procesamiento
            # - OCR para imágenes
            # - Extracción de texto de PDF
            # - Conversión a Markdown
            
            # Ejemplo simplificado:
            if self.file_type == self.FileType.PDF:
                self.markdown_content = extract_text_from_pdf(self.file.path)
            elif self.file_type == self.FileType.IMAGE:
                self.markdown_content = ocr_image(self.file.path)
            
            self.status = self.Status.COMPLETED
            self.save(update_fields=['status', 'markdown_content'])
            
        except Exception as e:
            self.status = self.Status.FAILED
            self.error_message = str(e)
            self.save(update_fields=['status', 'error_message'])
```

3. **Agregar método para generar respuesta de IA**:

```python
class Chat(models.Model):
    def ask(self, question: str) -> str:
        """Genera respuesta usando el contenido del documento como contexto."""
        # Crear mensaje del usuario
        Message.objects.create(
            chat=self,
            role=Message.Role.USER,
            content=question
        )
        
        # Preparar contexto para IA
        context = self.document.markdown_content
        history = self.messages.all()
        
        # Llamar a API de IA (OpenAI, Anthropic, etc.)
        response = call_ai_api(context, history, question)
        
        # Guardar respuesta
        Message.objects.create(
            chat=self,
            role=Message.Role.ASSISTANT,
            content=response
        )
        
        return response
```

4. **Agregar estadísticas de uso**:

```python
class UsageStats(models.Model):
    user = models.OneToOneField(settings.AUTH_USER_MODEL, on_delete=models.CASCADE)
    documents_count = models.PositiveIntegerField(default=0)
    total_storage_bytes = models.PositiveBigIntegerField(default=0)
    messages_count = models.PositiveIntegerField(default=0)
    updated_at = models.DateTimeField(auto_now=True)
    
    def update_stats(self):
        self.documents_count = Document.objects.filter(user=self.user).count()
        self.total_storage_bytes = Document.objects.filter(
            user=self.user
        ).aggregate(total=Sum('file_size'))['total'] or 0
        self.messages_count = Message.objects.filter(
            chat__user=self.user
        ).count()
        self.save()
```

---

### Containerización con Docker

**Archivo:** `docker-compose.yml`

Configuración con frontend React y backend Django separados.

**Recomendación:** Agregar servicio de Redis para caché y Celery para procesamiento asíncrono:

```yaml
services:
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
  
  celery:
    build: ./backend
    command: celery -A config worker -l info
    depends_on:
      - db
      - redis
    environment:
      - CELERY_BROKER_URL=redis://redis:6379/0
```

---

### Análisis Detallado de Arquitectura

**Este es el único proyecto con DIP (Inversión de Dependencias) real implementado.**

**Lo que funciona bien:**
- `AIService` como ABC con `@abstractmethod generate_response()` — DIP textbook
- `GeminiService` y `OpenAIService` como implementaciones concretas
- Factory Pattern (`get_ai_service()`) resuelve provider según settings
- `ChatMessage` con `tokens_used` — tracking de costos por mensaje
- Tests en `chats/tests.py`
- `.env.example` documentado

**Problemas arquitectónicos:**

1. **`GeminiService.__init__()` hace llamada a API externa en constructor:**

```python
class GeminiService(AIService):
    def __init__(self):
        genai.configure(api_key=settings.GEMINI_API_KEY)  # ← efecto secundario
        self.model = genai.GenerativeModel(...)            # ← inicialización costosa
```

Dificulta testing (no se puede instanciar sin credenciales). Usar lazy initialization:
```python
def _get_model(self):
    if not hasattr(self, '_model'):
        genai.configure(api_key=settings.GEMINI_API_KEY)
        self._model = genai.GenerativeModel(...)
    return self._model
```

2. **Factory con imports condicionales dentro de función:**
```python
def get_ai_service(provider):
    if provider == "openai":
        from .openai_service import OpenAIService  # ← import inside function
        return OpenAIService()
```
Mejor: registrar providers en un dict al inicio del módulo.

3. **`GeminiService` envía todo el historial sin límite** — a diferencia del patrón correcto que limita a últimos N mensajes. Tokens ilimitados = costos inesperados.


---

## Requisitos para Entrega 2 (Rúbrica)

### 1. Correcciones de la Parte 1 (10%)

- [ ] Diferir inicialización de `GeminiService` para evitar efecto secundario en `__init__`
- [ ] Registrar providers en un diccionario en vez de imports condicionales en factory
- [ ] Agregar límite de historial en `GeminiService` (como ya tiene `GroqService` del proyecto CatIA)

### 2. Diagrama de arquitectura actualizado (5%)

Entregar diagrama que refleje la arquitectura actual del sistema: capas (presentación, servicios, dominio, persistencia), componentes principales, y las interfaces abstractas (DIP). Formato legible (draw.io, Mermaid, PlantUML).

### 3. Servicios implementados (30%)

Ya tienen servicios de IA. Agregar: servicio de procesamiento de documentos (OCR/parsing), servicio de gestión de suscripciones

Cada servicio debe:
- Estar en un archivo `services.py` dentro de su app
- Encapsular la lógica de negocio (las vistas solo delegan)
- Usar `@transaction.atomic` donde haya operaciones de escritura múltiples

### 4. Inversión de dependencias (15%)

Ya tienen DIP implementado correctamente con `AIService` (ABC) + `GeminiService` + `OpenAIService`. Es el único proyecto con inversión de dependencias real.

Para la Entrega 2, asegurar que:
1. La factory (`get_ai_service()`) sea el único punto de creación
2. Ninguna vista importe directamente `GeminiService` u `OpenAIService`
3. Los tests usen un `MockAIService` que implemente `AIService`

### 5. Pruebas unitarias (10%)

Implementar al menos **dos pruebas unitarias** que verifiquen lógica de negocio:

```python
def test_gemini_service_genera_respuesta_con_contexto(self):
    # Mock de la API de Gemini → respuesta basada en documento

def test_factory_retorna_servicio_correcto_segun_settings(self):
    # AI_PROVIDER='openai' → isinstance(service, OpenAIService)
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
