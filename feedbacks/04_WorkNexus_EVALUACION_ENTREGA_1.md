# Evaluación Técnica - WorkNexus

---

## Análisis del Código Implementado

He revisado el código de su plataforma de conexión entre freelancers y clientes. El proyecto demuestra una arquitectura bien organizada con múltiples apps Django y modelos relacionales complejos.

### Modelos de Datos

**Archivos revisados:**
- `BACKEND/Projects/models.py` (89 líneas, 3 modelos)
- `BACKEND/Services/models.py` (18 líneas, 1 modelo)
- `BACKEND/Authentication/models.py` (perfiles de usuario)
- `BACKEND/Reviews/models.py` (sistema de reseñas)
- `BACKEND/Messaging/models.py` (mensajería)

Su modelado de datos es robusto y demuestra comprensión de relaciones complejas en Django.

**Modelo Project - Análisis detallado:**

```python
class Project(models.Model):
    CATEGORY_CHOICES = [
        ("desarrollo", "Desarrollo"),
        ("diseno", "Diseno"),
        ("marketing", "Marketing"),
        ("contenido", "Contenido"),
        ("soporte", "Soporte"),
        ("otros", "Otros"),
    ]
    STATUS_CHOICES = [
        ("abierto", "Abierto"),
        ("en_revision", "En revision"),
        ("en_ejecucion", "En ejecucion"),
        ("finalizado", "Finalizado"),
        ("cerrado", "Cerrado"),
    ]
```

**Fortalezas identificadas:**

1. **Uso correcto de choices**: Definieron constantes para categorías, estados y modalidades, lo cual es una excelente práctica.

2. **Campos bien pensados**: 
   - `budget` como DecimalField (correcto para dinero)
   - `deadline` como DateField
   - `modality` para remoto/híbrido/presencial
   - `is_open` como flag adicional al status

3. **Relaciones bien definidas**:
   - `client` → `ClientProfile` (ForeignKey con CASCADE)
   - `related_name='projects'` permite hacer `client.projects.all()`

4. **Meta class apropiada**: Ordenamiento por `-created_at` para mostrar proyectos más recientes primero.

**Modelo ProjectApplication - Excelente diseño:**

```python
class ProjectApplication(models.Model):
    project = models.ForeignKey(Project, on_delete=models.CASCADE, related_name="applications")
    freelancer = models.ForeignKey(FreelancerProfile, on_delete=models.CASCADE, related_name="project_applications")
    cover_letter = models.TextField(blank=True)
    proposed_budget = models.DecimalField(max_digits=10, decimal_places=2, blank=True, null=True)
    status = models.CharField(max_length=20, choices=STATUS_CHOICES, default="pendiente")
    
    class Meta:
        constraints = [
            models.UniqueConstraint(fields=["project", "freelancer"], name="unique_project_application"),
        ]
```

**Aspectos destacados:**

- **UniqueConstraint**: Previene que un freelancer aplique múltiples veces al mismo proyecto. Esto es **excelente** y demuestra comprensión de integridad de datos.
- **proposed_budget**: Permite que el freelancer proponga su propio presupuesto, buen diseño de negocio.
- **cover_letter**: Campo para carta de presentación.

**Modelo ProjectFavorite - Sistema de favoritos:**

Implementaron un sistema de favoritos que permite a freelancers marcar proyectos de interés. También usa `UniqueConstraint` para evitar duplicados.

**Áreas de mejora importantes:**

1. **Falta validación de presupuesto positivo**:

```python
from django.core.validators import MinValueValidator

class Project(models.Model):
    budget = models.DecimalField(
        max_digits=10, 
        decimal_places=2,
        validators=[MinValueValidator(0.01)]  # Presupuesto debe ser positivo
    )
```

2. **Falta validación de deadline futuro**:

```python
from django.core.exceptions import ValidationError
from django.utils import timezone

class Project(models.Model):
    # ... campos existentes ...
    
    def clean(self):
        if self.deadline and self.deadline < timezone.now().date():
            raise ValidationError('La fecha límite debe ser futura')
```

3. **El modelo Service está muy simple**. Debería tener más campos:

```python
class Service(models.Model):
    title = models.CharField(max_length=255)
    description = models.TextField()
    category = models.CharField(max_length=50, choices=CATEGORY_CHOICES)
    freelancer = models.ForeignKey(FreelancerProfile, on_delete=models.CASCADE, related_name="services")
    
    # AGREGAR:
    price = models.DecimalField(max_digits=10, decimal_places=2, validators=[MinValueValidator(0)])
    delivery_time = models.PositiveIntegerField(help_text="Días de entrega")
    is_active = models.BooleanField(default=True)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
```

4. **Falta método de negocio en Project**:

```python
class Project(models.Model):
    # ... campos existentes ...
    
    def close_project(self):
        """Cierra el proyecto y marca como no disponible"""
        if self.status != 'finalizado':
            raise ValueError('Solo se pueden cerrar proyectos finalizados')
        self.status = 'cerrado'
        self.is_open = False
        self.save(update_fields=['status', 'is_open'])
    
    def accept_application(self, application):
        """Acepta una aplicación y rechaza las demás"""
        if self.status != 'abierto':
            raise ValueError('El proyecto ya no está abierto')
        
        application.status = 'aceptada'
        application.save()
        
        # Rechazar otras aplicaciones
        self.applications.exclude(id=application.id).update(status='rechazada')
        
        # Cambiar estado del proyecto
        self.status = 'en_ejecucion'
        self.is_open = False
        self.save(update_fields=['status', 'is_open'])
```

---

### Arquitectura de Apps

**Excelente separación de responsabilidades:**

- `Authentication`: Manejo de usuarios, perfiles de clientes y freelancers
- `Projects`: Proyectos, aplicaciones y favoritos
- `Services`: Servicios ofrecidos por freelancers
- `Reviews`: Sistema de reseñas
- `Messaging`: Sistema de mensajería
- `order`: Órdenes/transacciones

Esta arquitectura modular facilita el mantenimiento y escalabilidad del proyecto.

---

### Containerización con Docker

**Archivo:** `docker-compose.yml`

Configuración correcta con:
- Servicio PostgreSQL (puerto 5432)
- Backend Django (puerto 8000)
- Frontend (puerto 8080)
- Volúmenes para persistencia de datos

**Observación:** No encontré healthcheck en la base de datos. Agregar:

```yaml
services:
  db:
    image: postgres:15
    environment:
      POSTGRES_DB: WorkNexus
      POSTGRES_USER: jose_anna
      POSTGRES_PASSWORD: password123
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U jose_anna"]
      interval: 5s
      timeout: 5s
      retries: 5

  backend:
    depends_on:
      db:
        condition: service_healthy  # Esperar a que DB esté lista
```

---

### Análisis Detallado de Vistas y Seguridad

**Problemas críticos:**

1. **`@csrf_exempt` en endpoints de autenticación — riesgo de seguridad grave:**

```python
@method_decorator(csrf_exempt, name="dispatch")
class RegisterView(View):    # ← Registro sin protección CSRF
@method_decorator(csrf_exempt, name="dispatch")
class LoginView(View):       # ← Login sin protección CSRF
```

Un atacante puede crear cuentas o hacer login en nombre de otro usuario desde un sitio malicioso. Si el proyecto es una API REST, deben usar Django REST Framework con JWT, que maneja CSRF correctamente.

2. **Parsing manual de JSON en lugar de usar DRF:**

```python
# Actual — frágil, verboso, sin estándar
data = json.loads(request.body)
email = data.get("email")
if not email or not password:
    return JsonResponse({"error": "..."}, status=400)
```

DRF ofrece serialización, validación y manejo de errores estandarizados. La validación manual es propensa a errores y no genera mensajes de error consistentes.

3. **Imports dentro de métodos:**

```python
def post(self, request):
    from django.contrib.auth import get_user_model  # ← debe ir al top
    from .models import ClientProfile, FreelancerProfile  # ← debe ir al top
```

Esto afecta legibilidad y hace que Python re-evalúe el import en cada request.

4. **`age` como `PositiveIntegerField` en `FreelancerProfile`:**

```python
age = models.PositiveIntegerField()  # Envejece los datos
# Correcto: date_of_birth = models.DateField()
```

La edad cambia cada año. Se debe guardar `fecha_nacimiento` y calcular la edad dinámicamente.

5. **Sin capa de servicios.** La lógica de registro (crear usuario + crear perfil + login) está en 80+ líneas dentro de la vista. Debe abstraerse a un `UserRegistrationService`.

6. **Sin tests visibles.** No hay ningún test para autenticación, registro o flujos de negocio.


---

## Requisitos para Entrega 2 (Rúbrica)

### 1. Correcciones de la Parte 1 (10%)

- [ ] Eliminar `@csrf_exempt` de todos los endpoints de autenticación
- [ ] Migrar de JSON manual a Django REST Framework con serializers
- [ ] Mover imports al top de los archivos
- [ ] Cambiar `age` por `date_of_birth` en `FreelancerProfile`
- [ ] Crear capa de servicios para lógica de negocio

### 2. Diagrama de arquitectura actualizado (5%)

Entregar diagrama que refleje la arquitectura actual del sistema: capas (presentación, servicios, dominio, persistencia), componentes principales, y las interfaces abstractas (DIP). Formato legible (draw.io, Mermaid, PlantUML).

### 3. Servicios implementados (30%)

Servicio de registro/autenticación (usar DRF + JWT), servicio de matching freelancer-cliente, servicio de gestión de proyectos

Cada servicio debe:
- Estar en un archivo `services.py` dentro de su app
- Encapsular la lógica de negocio (las vistas solo delegan)
- Usar `@transaction.atomic` donde haya operaciones de escritura múltiples

### 4. Inversión de dependencias (15%)

Crear una interfaz abstracta para el sistema de mensajería:

```python
# messaging/base.py
from abc import ABC, abstractmethod

class MessagingService(ABC):
    @abstractmethod
    def send_message(self, sender, recipient, content) -> dict: ...
    
    @abstractmethod
    def get_conversation(self, user1, user2) -> list: ...

# messaging/db_messaging.py
class DatabaseMessagingService(MessagingService):
    def send_message(self, sender, recipient, content) -> dict:
        return Message.objects.create(sender=sender, recipient=recipient, content=content)

# messaging/websocket_messaging.py  
class WebSocketMessagingService(MessagingService):
    def send_message(self, sender, recipient, content) -> dict:
        # Enviar via WebSocket + guardar en DB
        ...
```

### 5. Pruebas unitarias (10%)

Implementar al menos **dos pruebas unitarias** que verifiquen lógica de negocio:

```python
def test_registro_freelancer_crea_perfil(self):
    # POST /register con userType=freelancer → FreelancerProfile existe

def test_cliente_no_puede_ver_panel_freelancer(self):
    # Login como cliente → acceso a endpoints de freelancer denegado
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
