# Evaluación Técnica - Pragma

---

## Estado Actual del Proyecto

El repositorio registrado (`dcossios/pragma`) no existe o es privado en GitHub. No fue posible acceder al código fuente para realizar la evaluación técnica.

La nota se ajusta a 3.0 por política del curso. Sin embargo, para la segunda entrega el repositorio **debe ser público y accesible** con una implementación sustancial.

---

## Requisitos para la Segunda Entrega

A continuación se detalla exhaustivamente lo que se debe implementar. El dominio es una plataforma de gestión de proyectos (similar a Jira/Trello con enfoque en equipos de software).

---

### 1. Repositorio y Configuración Base (URGENTE)

Antes de escribir una línea de código de negocio:

**Hacer el repositorio público:**
```
GitHub → Settings → Danger Zone → Change visibility → Public
```

**Variables de entorno — nunca hardcodear secretos:**
```python
# settings.py
import os
SECRET_KEY  = os.environ['DJANGO_SECRET_KEY']
DEBUG       = os.getenv('DEBUG', 'False') == 'True'
ALLOWED_HOSTS = os.getenv('ALLOWED_HOSTS', 'localhost').split(',')

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': os.environ['DB_NAME'],
        'USER': os.environ['DB_USER'],
        'PASSWORD': os.environ['DB_PASSWORD'],
        'HOST': os.getenv('DB_HOST', 'localhost'),
    }
}
```

**`.gitignore` con:**
```
.env
__pycache__/
*.pyc
db.sqlite3
media/
```

---

### 2. Modelos de Dominio (OBLIGATORIO)

```python
# projects/models.py
import uuid
from django.db import models
from django.conf import settings

class Project(models.Model):
    id          = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
    name        = models.CharField(max_length=200)
    description = models.TextField(blank=True)
    owner       = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.CASCADE,
                                    related_name='owned_projects')
    members     = models.ManyToManyField(settings.AUTH_USER_MODEL, related_name='projects', blank=True)
    created_at  = models.DateTimeField(auto_now_add=True)
    updated_at  = models.DateTimeField(auto_now=True)

class Task(models.Model):
    class Status(models.TextChoices):
        TODO        = 'todo',        'Por hacer'
        IN_PROGRESS = 'in_progress', 'En progreso'
        REVIEW      = 'review',      'En revisión'
        DONE        = 'done',        'Hecho'

    class Priority(models.TextChoices):
        LOW    = 'low',    'Baja'
        MEDIUM = 'medium', 'Media'
        HIGH   = 'high',   'Alta'

    id          = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
    project     = models.ForeignKey(Project, on_delete=models.CASCADE, related_name='tasks')
    title       = models.CharField(max_length=300)
    description = models.TextField(blank=True)
    assignee    = models.ForeignKey(settings.AUTH_USER_MODEL, null=True, blank=True,
                                    on_delete=models.SET_NULL, related_name='assigned_tasks')
    status      = models.CharField(max_length=20, choices=Status.choices, default=Status.TODO)
    priority    = models.CharField(max_length=10, choices=Priority.choices, default=Priority.MEDIUM)
    due_date    = models.DateField(null=True, blank=True)
    created_at  = models.DateTimeField(auto_now_add=True)
    updated_at  = models.DateTimeField(auto_now=True)
```

---

### 3. Capa de Servicios (OBLIGATORIO para esta asignatura)

La lógica de negocio nunca va en las vistas:

```python
# projects/services.py
from django.db import transaction
from .models import Task, Project

class TaskService:
    @staticmethod
    def move_task(task: Task, new_status: str, acting_user) -> Task:
        """Cambia el estado de una tarea. Solo miembros del proyecto pueden hacerlo."""
        if acting_user not in task.project.members.all() and task.project.owner != acting_user:
            raise PermissionError("No eres miembro de este proyecto.")

        valid_statuses = {s.value for s in Task.Status}
        if new_status not in valid_statuses:
            raise ValueError(f"Estado inválido: {new_status}")

        task.status = new_status
        task.save(update_fields=['status', 'updated_at'])
        return task

    @staticmethod
    @transaction.atomic
    def assign_task(task: Task, assignee, acting_user) -> Task:
        """Asigna una tarea a un miembro del proyecto."""
        if assignee not in task.project.members.all():
            raise ValueError("El asignado debe ser miembro del proyecto.")
        task.assignee = assignee
        task.save(update_fields=['assignee', 'updated_at'])
        return task
```

---

### 4. Endpoints REST (OBLIGATORIO)

| Método | URL | Descripción |
|--------|-----|-------------|
| POST | `/api/auth/register/` | Registro |
| POST | `/api/auth/login/` | Login → JWT |
| GET/POST | `/api/projects/` | Listar / crear proyectos |
| GET/PUT/DELETE | `/api/projects/{id}/` | Detalle / editar / eliminar |
| POST | `/api/projects/{id}/members/` | Agregar miembro |
| GET/POST | `/api/projects/{id}/tasks/` | Listar / crear tareas |
| PATCH | `/api/tasks/{id}/status/` | Cambiar estado de tarea |
| PATCH | `/api/tasks/{id}/assign/` | Asignar tarea |

---

### 5. Docker Funcional (OBLIGATORIO)

```dockerfile
# Dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 8000
CMD ["python", "manage.py", "runserver", "0.0.0.0:8000"]
```

```yaml
# docker-compose.yml
services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: pragma_db
      POSTGRES_USER: pragma_user
      POSTGRES_PASSWORD: pragma_pass
    volumes:
      - postgres_data:/var/lib/postgresql/data

  web:
    build: .
    ports:
      - "8000:8000"
    environment:
      DJANGO_SECRET_KEY: change-me-in-production
      DEBUG: "True"
      DB_NAME: pragma_db
      DB_USER: pragma_user
      DB_PASSWORD: pragma_pass
      DB_HOST: db
    depends_on:
      - db

volumes:
  postgres_data:
```

---

### 6. Tests (OBLIGATORIO)

```python
class TaskServiceTests(TestCase):
    def test_solo_miembro_puede_mover_tarea(self): ...
    def test_estado_invalido_lanza_excepcion(self): ...
    def test_asignado_debe_ser_miembro(self): ...
    def test_tarea_cambia_estado_correctamente(self): ...
    def test_listado_solo_muestra_tareas_del_proyecto(self): ...
```

---

### 7. Buenas Prácticas de Git

- Commits frecuentes con mensajes descriptivos
- Nunca subir `.env`, `db.sqlite3`, `__pycache__`
- Usar ramas por feature: `feat/task-service`, `feat/auth`, etc.

---

## Resumen de Prioridades

| Prioridad | Tarea |
|:---------:|-------|
| 🔴 Crítico | Hacer el repositorio público |
| 🔴 Crítico | Docker funcional |
| 🔴 Crítico | Variables de entorno para secretos |
| 🟠 Alto | Modelos de dominio completos |
| 🟠 Alto | Endpoints REST funcionales |
| 🟠 Alto | Capa de servicios |
| 🟡 Medio | Tests |
| 🟡 Medio | README con instrucciones |

---

*No fue posible acceder al repositorio `dcossios/pragma`. Si el proyecto existe y es privado, debe hacerse público para poder evaluarlo en la segunda entrega.*
