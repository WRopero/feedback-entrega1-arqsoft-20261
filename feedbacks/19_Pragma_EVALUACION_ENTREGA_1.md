# Evaluación Técnica - Pragma

## Nota Final: 0.51/5.0

---

## Análisis del Código Implementado

He revisado el repositorio de su proyecto y **no encontré implementación de código Django**.

### Estado Actual:

El repositorio no contiene archivos de código fuente. Esto indica que el proyecto está en fase muy inicial o no se subió correctamente al repositorio.

### Requisitos Mínimos No Cumplidos:

1. **No hay proyecto Django** (manage.py, settings.py, etc.)
2. **No hay modelos de datos**
3. **No hay vistas implementadas**
4. **No hay Docker configurado**
5. **No hay README**

### Acciones Urgentes Requeridas:

Esta situación es **crítica**. Para la segunda entrega debe:

**1. Crear proyecto Django desde cero:**

```bash
django-admin startproject pragma
cd pragma
python manage.py startapp core
```

**2. Implementar modelos según su dominio de negocio:**

```python
# core/models.py
from django.db import models

class Entidad1(models.Model):
    # Definir campos según su negocio
    nombre = models.CharField(max_length=200)
    descripcion = models.TextField()
    created_at = models.DateTimeField(auto_now_add=True)

class Entidad2(models.Model):
    # Relación con Entidad1
    entidad1 = models.ForeignKey(Entidad1, on_delete=models.CASCADE)
    # Otros campos
```

**3. Crear vistas con Django REST Framework:**

```python
# core/views.py
from rest_framework import generics
from .models import Entidad1
from .serializers import Entidad1Serializer

class Entidad1ListView(generics.ListCreateAPIView):
    queryset = Entidad1.objects.all()
    serializer_class = Entidad1Serializer
```

**4. Configurar Docker:**

```yaml
# docker-compose.yml
version: '3.8'

services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: miapp_db
      POSTGRES_USER: miapp_user
      POSTGRES_PASSWORD: miapp_pass
    volumes:
      - postgres_data:/var/lib/postgresql/data

  web:
    build: .
    command: python manage.py runserver 0.0.0.0:8000
    volumes:
      - .:/app
    ports:
      - "8000:8000"
    depends_on:
      - db

volumes:
  postgres_data:
```

**5. Crear README con instrucciones:**

```markdown
# Pragma

## Descripción
[Describir el proyecto]

## Instalación

1. Clonar repositorio
2. Ejecutar: `docker-compose up --build`
3. Acceder a: http://localhost:8000

## Funcionalidades
[Listar funcionalidades implementadas]
```

---

## Recomendación Final

Deben ponerse al día urgentemente. La segunda entrega requiere:

1. Proyecto Django completo y funcional
2. Docker configurado correctamente
3. Modelos, vistas y serializers implementados
4. Al menos 3-5 funcionalidades core funcionando
5. README con documentación completa
6. Demo funcional para la sustentación

Consideren buscar ayuda del profesor o compañeros para ponerse al día rápidamente.


### Buenas Prácticas de Git y Control de Versiones

**Problema Crítico:** No se encontró repositorio Git en el proyecto.

El control de versiones con Git es **fundamental** para cualquier proyecto de software profesional. Para la segunda entrega deben:

1. **Inicializar Git en el proyecto:**
   ```bash
   git init
   git add .
   git commit -m "feat: initial commit"
   ```

2. **Configurar usuario:**
   ```bash
   git config user.name "Nombre Completo"
   git config user.email "email@eafit.edu.co"
   ```

3. **Hacer commits frecuentes** con mensajes descriptivos:
   ```bash
   git add archivo.py
   git commit -m "feat: agregar modelo de Usuario"
   ```

4. **Usar Conventional Commits:**
   - `feat:` para nuevas funcionalidades
   - `fix:` para correcciones
   - `docs:` para documentación
   - `refactor:` para mejoras de código

5. **Subir a GitHub:**
   ```bash
   git remote add origin https://github.com/usuario/proyecto.git
   git push -u origin main
   ```

**Sin Git, el proyecto pierde trazabilidad y colaboración efectiva.**

---
---

*Evaluación realizada sobre el código fuente del repositorio GitHub.*
