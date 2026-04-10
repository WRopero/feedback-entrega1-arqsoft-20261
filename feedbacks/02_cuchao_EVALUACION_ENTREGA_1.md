# Evaluación Técnica - Cuchao

---

## Análisis del Código Implementado

He revisado el código de su plataforma de compra y venta de carros usados. El proyecto demuestra una implementación sólida de Django tradicional con templates, siguiendo correctamente el patrón MVT.

### Modelos de Datos

**Archivo:** `carros/models.py` (62 líneas, 3 modelos)

Implementaron un modelado simple pero funcional:

**Usuario**: Modelo personalizado basado en `AbstractUser`. Esto es una buena práctica porque permite extender el modelo de usuario en el futuro.

```python
class Usuario(AbstractUser):
    pass
```

**Carro**: Modelo principal con los campos esenciales para un vehículo:
- `propietario`: ForeignKey a Usuario con `null=True, blank=True`
- `modelo`, `descripcion`, `imagen`, `precio`
- `vendido`: Boolean para controlar disponibilidad
- Método `esta_disponible()` que retorna el estado de venta

**Compra**: Modelo para registro de transacciones con:
- Relaciones a Usuario y Carro
- `METODOS_PAGO` como choices (efectivo, PSE)
- `precio_pagado` para snapshot del precio
- `fecha_compra` con auto_now_add
- Meta ordering por fecha descendente

**Áreas de mejora importantes:**

1. **Falta validación de precio positivo**. Actualmente el modelo acepta precios negativos:

```python
# Mejora sugerida
from django.core.validators import MinValueValidator

class Carro(models.Model):
    precio = models.DecimalField(
        max_digits=10, 
        decimal_places=2,
        validators=[MinValueValidator(0.01)]  # Precio debe ser positivo
    )
```

2. **El campo `propietario` permite NULL**, lo cual no tiene sentido para un carro publicado:

```python
# Debería ser:
propietario = models.ForeignKey(
    Usuario,
    on_delete=models.CASCADE,  # Si se elimina usuario, eliminar sus carros
    related_name='carros_publicados'
)
```

3. **Falta validación de que el comprador no sea el propietario**. Esto debería validarse en el modelo:

```python
class Compra(models.Model):
    def clean(self):
        if self.comprador == self.carro.propietario:
            raise ValidationError('No puedes comprar tu propio carro')
```

4. **Agregar timestamps para auditoría**:

```python
class Carro(models.Model):
    # ... campos existentes ...
    fecha_publicacion = models.DateTimeField(auto_now_add=True)
    fecha_actualizacion = models.DateTimeField(auto_now=True)
```

---

### Vistas y Lógica de Negocio

**Archivo:** `carros/views.py` (177 líneas, 10 funciones)

Implementaron vistas basadas en funciones (function-based views) con Django tradicional. Esto es válido pero menos escalable que usar class-based views o APIs REST.

**Aspectos positivos:**

1. **Uso de decoradores apropiados**:
   - `@login_required` para proteger rutas
   - `@require_http_methods` para limitar métodos HTTP

2. **Validación de permisos**: Verifican que solo el propietario pueda editar/eliminar:

```python
if carro.propietario != request.user:
    return HttpResponseForbidden('No tienes permiso para editar este carro')
```

3. **Flujo de compra bien implementado**: Separación entre confirmación y procesamiento de compra.

4. **Validación de estado**: Verifican que el carro no esté vendido antes de procesar compra.

**Problemas críticos encontrados:**

1. **Falta transacción atómica en `procesar_compra`**. Si falla el `save()` del carro después de crear la compra, quedaría inconsistencia:

```python
from django.db import transaction

@login_required(login_url='login')
@require_http_methods(["POST"])
@transaction.atomic  # AGREGAR ESTO
def procesar_compra(request, carro_id):
    carro = get_object_or_404(Carro, id=carro_id)
    
    # Usar select_for_update para evitar race conditions
    carro = Carro.objects.select_for_update().get(id=carro_id)
    
    if carro.vendido:
        messages.error(request, 'Este carro ya ha sido vendido.')
        return redirect('catalogo')
    
    # ... resto del código
```

2. **Race condition en compras simultáneas**. Dos usuarios podrían comprar el mismo carro al mismo tiempo. Solución:

```python
from django.db import transaction

@transaction.atomic
def procesar_compra(request, carro_id):
    # Lock del registro para evitar compras simultáneas
    carro = Carro.objects.select_for_update().get(id=carro_id)
    
    if carro.vendido:
        messages.error(request, 'Este carro ya ha sido vendido.')
        return redirect('catalogo')
    
    # ... crear compra y marcar como vendido
```

3. **La vista `catalogo` muestra TODOS los carros sin filtrar**:

```python
def catalogo(request):
    carros = Carro.objects.all()  # Muestra vendidos y no vendidos
    return render(request, 'catalogo.html', {'carros': carros})
```

Debería filtrar o al menos separar:

```python
def catalogo(request):
    carros_disponibles = Carro.objects.filter(vendido=False).order_by('-id')
    carros_vendidos = Carro.objects.filter(vendido=True).order_by('-id')
    
    context = {
        'carros_disponibles': carros_disponibles,
        'carros_vendidos': carros_vendidos,
    }
    return render(request, 'catalogo.html', context)
```

4. **Falta validación de que el usuario no compre su propio carro**:

```python
@transaction.atomic
def procesar_compra(request, carro_id):
    carro = Carro.objects.select_for_update().get(id=carro_id)
    
    # AGREGAR ESTA VALIDACIÓN
    if carro.propietario == request.user:
        messages.error(request, 'No puedes comprar tu propio carro.')
        return redirect('catalogo')
    
    # ... resto del código
```

5. **No hay manejo de errores en autenticación**:

```python
def login_view(request):
    if request.method == 'POST':
        username = request.POST.get('username', '')
        password = request.POST.get('password', '')
        
        # Falta validación de intentos fallidos
        # Falta protección contra brute force
        
        user = authenticate(request, username=username, password=password)
```

Mejora sugerida:

```python
from django.core.cache import cache

def login_view(request):
    if request.method == 'POST':
        username = request.POST.get('username', '')
        
        # Protección contra brute force
        attempts_key = f'login_attempts_{username}'
        attempts = cache.get(attempts_key, 0)
        
        if attempts >= 5:
            return render(request, 'login.html', {
                'error': 'Demasiados intentos fallidos. Intenta en 15 minutos.'
            })
        
        user = authenticate(request, username=username, password=password)
        if user is not None:
            cache.delete(attempts_key)
            login(request, user)
            return redirect('catalogo')
        else:
            cache.set(attempts_key, attempts + 1, 900)  # 15 minutos
            return render(request, 'login.html', {
                'error': 'Usuario o contraseña incorrectos'
            })
```

---

### Templates

Implementaron 8 templates HTML con estructura base. Esto es correcto para un proyecto Django tradicional.

**Observaciones:**

- Usan `base.html` para herencia de templates (buena práctica)
- Templates específicos para cada funcionalidad
- Falta implementar mensajes de Django (`{% if messages %}`) en base.html para mostrar feedback al usuario

**Mejora sugerida en base.html:**

```html
{% if messages %}
<div class="messages">
    {% for message in messages %}
    <div class="alert alert-{{ message.tags }}">
        {{ message }}
    </div>
    {% endfor %}
</div>
{% endif %}
```

---

### Containerización con Docker

**Archivo:** `docker-compose.yml`

Configuración correcta con:
- Servicio PostgreSQL con healthcheck
- Servicio web con Django
- Volúmenes para persistencia
- Variables de entorno

**Mejora sugerida:**

Agregar volumen para media files:

```yaml
services:
  web:
    volumes:
      - .:/app
      - media_files:/app/media  # Para persistir imágenes

volumes:
  postgres_data:
  media_files:  # Agregar este volumen
```
