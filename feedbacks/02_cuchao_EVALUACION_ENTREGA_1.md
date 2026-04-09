# Evaluación Técnica - Cuchao

## Nota Final: 4.49/5.0

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

---

### Buenas Prácticas de Git y Control de Versiones

**Análisis del historial de commits:** 12 commits totales

**Aspectos positivos:**

- **Historial moderado**: 12 commits es un número razonable para el alcance del proyecto.
- **Mensajes descriptivos**: 100% de los commits tienen mensajes claros. Ejemplos:
  - "Agregué funcionalidad de compra simulada"
  - "Update README.md"
- **Equipo consistente**: 3 autores trabajando en el proyecto, lo cual indica colaboración efectiva.
- **Commits rastreables**: Los cambios están bien documentados en el historial.

**Áreas de mejora:**

1. **Evitar mensajes genéricos**: Algunos commits como "Add files via upload" o "mis cambios antes de pull" no son descriptivos. Mejor usar:
   ```
   feat: agregar modelo de Compra para historial de transacciones
   fix: corregir validación de stock en proceso de compra
   docs: actualizar README con instrucciones de Docker
   ```

2. **Commits más frecuentes**: Con solo 12 commits, parece que agruparon muchos cambios en pocos commits. Es mejor hacer commits más pequeños y frecuentes.

**Recomendaciones para la segunda entrega:**

1. **Adoptar Conventional Commits**:
   - `feat:` nuevas funcionalidades
   - `fix:` correcciones
   - `refactor:` mejoras de código
   - `docs:` documentación

2. **Commits atómicos**: Cada commit debe tener un propósito único y claro.

3. **Mensajes en español o inglés consistentemente**: Evitar mezclar idiomas en los mensajes.

---

## Recomendaciones Específicas para la Segunda Entrega

### 1. Migrar a Django REST Framework

Su proyecto actual usa templates tradicionales. Para modernizarlo y facilitar integración con frontends:

```python
# views.py con DRF
from rest_framework import viewsets, permissions
from rest_framework.decorators import action
from rest_framework.response import Response

class CarroViewSet(viewsets.ModelViewSet):
    queryset = Carro.objects.all()
    serializer_class = CarroSerializer
    permission_classes = [permissions.IsAuthenticatedOrReadOnly]
    
    def get_queryset(self):
        if self.request.query_params.get('disponibles'):
            return Carro.objects.filter(vendido=False)
        return super().get_queryset()
    
    @action(detail=True, methods=['post'])
    def comprar(self, request, pk=None):
        carro = self.get_object()
        
        if carro.vendido:
            return Response(
                {'error': 'Carro ya vendido'},
                status=400
            )
        
        if carro.propietario == request.user:
            return Response(
                {'error': 'No puedes comprar tu propio carro'},
                status=400
            )
        
        with transaction.atomic():
            carro = Carro.objects.select_for_update().get(pk=pk)
            
            compra = Compra.objects.create(
                comprador=request.user,
                carro=carro,
                metodo_pago=request.data.get('metodo_pago'),
                precio_pagado=carro.precio
            )
            
            carro.vendido = True
            carro.save()
        
        return Response(CompraSerializer(compra).data)
```

### 2. Implementar Búsqueda y Filtros

```python
from django.db.models import Q

def catalogo(request):
    carros = Carro.objects.filter(vendido=False)
    
    # Búsqueda por modelo
    search = request.GET.get('search')
    if search:
        carros = carros.filter(
            Q(modelo__icontains=search) | 
            Q(descripcion__icontains=search)
        )
    
    # Filtro por rango de precio
    precio_min = request.GET.get('precio_min')
    precio_max = request.GET.get('precio_max')
    
    if precio_min:
        carros = carros.filter(precio__gte=precio_min)
    if precio_max:
        carros = carros.filter(precio__lte=precio_max)
    
    return render(request, 'catalogo.html', {'carros': carros})
```

### 3. Agregar Sistema de Favoritos

```python
# models.py
class Favorito(models.Model):
    usuario = models.ForeignKey(Usuario, on_delete=models.CASCADE)
    carro = models.ForeignKey(Carro, on_delete=models.CASCADE)
    fecha_agregado = models.DateTimeField(auto_now_add=True)
    
    class Meta:
        unique_together = ('usuario', 'carro')
```

### 4. Implementar Notificaciones por Email

```python
from django.core.mail import send_mail

def notificar_venta(compra):
    # Notificar al vendedor
    send_mail(
        subject=f'¡Vendiste tu {compra.carro.modelo}!',
        message=f'Tu carro ha sido comprado por ${compra.precio_pagado}',
        from_email='noreply@cuchao.com',
        recipient_list=[compra.carro.propietario.email],
    )
    
    # Notificar al comprador
    send_mail(
        subject=f'Compra confirmada: {compra.carro.modelo}',
        message=f'Has comprado {compra.carro.modelo} por ${compra.precio_pagado}',
        from_email='noreply@cuchao.com',
        recipient_list=[compra.comprador.email],
    )
```

### 5. Agregar Historial de Compras y Ventas

```python
@login_required
def mis_compras(request):
    compras = Compra.objects.filter(
        comprador=request.user
    ).select_related('carro').order_by('-fecha_compra')
    
    return render(request, 'mis_compras.html', {'compras': compras})

@login_required
def mis_ventas(request):
    ventas = Compra.objects.filter(
        carro__propietario=request.user
    ).select_related('carro', 'comprador').order_by('-fecha_compra')
    
    return render(request, 'mis_ventas.html', {'ventas': ventas})
```

### 6. Mejorar Seguridad

```python
# settings.py
SECURE_BROWSER_XSS_FILTER = True
SECURE_CONTENT_TYPE_NOSNIFF = True
X_FRAME_OPTIONS = 'DENY'
CSRF_COOKIE_SECURE = True  # En producción
SESSION_COOKIE_SECURE = True  # En producción

# Agregar validación de imágenes
from PIL import Image

def validar_imagen(imagen):
    try:
        img = Image.open(imagen)
        img.verify()
        return True
    except:
        return False
```

---

## Aspectos Positivos del Proyecto

- Implementación completa del flujo de compra/venta
- Docker configurado correctamente
- Separación de responsabilidades en vistas
- Control de permisos implementado
- README muy completo y detallado
- Modelo de Usuario personalizado (buena práctica)
- Templates organizados con herencia
- Uso de mensajes de Django para feedback

---

## Próximos Pasos

1. **Corregir el race condition** en compras usando `select_for_update()`
2. **Agregar transacciones atómicas** en operaciones críticas
3. **Implementar búsqueda y filtros** en el catálogo
4. **Migrar a API REST** para mayor flexibilidad
5. **Agregar tests** para las funcionalidades críticas
6. **Implementar sistema de favoritos** y notificaciones
7. **Mejorar validaciones** en modelos y vistas

Su proyecto tiene una base sólida. La arquitectura MVT está bien implementada y el flujo de negocio es correcto. Para la segunda entrega, enfóquense en corregir los problemas de concurrencia, agregar las funcionalidades sugeridas y mejorar la experiencia de usuario con búsqueda y filtros.

---

*Evaluación realizada sobre el código fuente del repositorio GitHub.*
