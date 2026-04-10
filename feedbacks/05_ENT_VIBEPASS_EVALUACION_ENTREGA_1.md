# Evaluación Técnica - ENT VIBEPASS

---

## Análisis del Código Implementado

He revisado el código de su plataforma de venta de tickets para eventos. El proyecto demuestra una implementación sólida de Django con templates, modelos bien estructurados y vistas optimizadas.

### Modelos de Datos

**Archivos revisados:**
- `apps/eventos/models.py` (49 líneas, 4 modelos)
- `apps/reservas/models.py` (32 líneas, 2 modelos)
- `apps/pagos/models.py` (27 líneas, 1 modelo)
- `apps/usuarios/models.py` (perfiles de usuario)

Su modelado de datos es limpio y bien pensado para un sistema de ticketing.

**Modelo Evento - Análisis detallado:**

```python
class Evento(models.Model):
    nombre = models.CharField(max_length=150)
    descripcion = models.TextField()
    fecha = models.DateField()
    hora = models.TimeField()
    capacidad = models.PositiveIntegerField(default=0)
    organizador = models.CharField(max_length=150, default='')
    imagen = models.ImageField(upload_to='eventos/', blank=True, null=True)
    categoria = models.ForeignKey(CategoriaEvento, on_delete=models.CASCADE, related_name='eventos')
    lugar = models.ForeignKey(Lugar, on_delete=models.CASCADE, related_name='eventos')
```

**Fortalezas identificadas:**

1. **Separación de fecha y hora**: Usar `DateField` y `TimeField` separados es correcto para eventos.

2. **Modelo Lugar bien diseñado**:
```python
class Lugar(models.Model):
    nombre = models.CharField(max_length=150)
    direccion = models.CharField(max_length=200)
    ciudad = models.CharField(max_length=100)
    capacidad = models.PositiveIntegerField()
    latitud = models.FloatField(default=0.0)
    longitud = models.FloatField(default=0.0)
```
- Incluye coordenadas GPS (latitud/longitud) para mapas
- Campo de capacidad para validar aforo

3. **Sistema de tipos de ticket flexible**:
```python
class TipoTicket(models.Model):
    evento = models.ForeignKey(Evento, on_delete=models.CASCADE, related_name='tipos_ticket')
    nombre = models.CharField(max_length=100)
    precio = models.DecimalField(max_digits=10, decimal_places=2)
    cantidad_disponible = models.PositiveIntegerField()
```
- Permite múltiples tipos de tickets por evento (VIP, General, etc.)
- Control de disponibilidad por tipo

4. **Modelo Reserva con estados**:
```python
class Reserva(models.Model):
    ESTADOS = [
        ('pendiente', 'Pendiente'),
        ('confirmada', 'Confirmada'),
        ('cancelada', 'Cancelada'),
    ]
    usuario = models.ForeignKey(User, on_delete=models.CASCADE, related_name='reservas')
    evento = models.ForeignKey(Evento, on_delete=models.CASCADE, related_name='reservas')
    tipo_ticket = models.ForeignKey(TipoTicket, on_delete=models.CASCADE, related_name='reservas')
    cantidad = models.PositiveIntegerField()
    estado = models.CharField(max_length=20, choices=ESTADOS, default='pendiente')
```

5. **Modelo Ticket individual**:
```python
class Ticket(models.Model):
    reserva = models.ForeignKey(Reserva, on_delete=models.CASCADE, related_name='tickets')
    codigo = models.CharField(max_length=100, unique=True)
    precio_final = models.DecimalField(max_digits=10, decimal_places=2)
    usado = models.BooleanField(default=False)
```
- Código único por ticket (para validación en entrada)
- Flag `usado` para control de acceso
- Snapshot del precio en `precio_final`

6. **Sistema de pagos**:
```python
class Pago(models.Model):
    METODOS = [
        ('tarjeta', 'Tarjeta'),
        ('pse', 'PSE'),
        ('efectivo', 'Efectivo'),
    ]
    ESTADOS = [
        ('pendiente', 'Pendiente'),
        ('aprobado', 'Aprobado'),
        ('rechazado', 'Rechazado'),
    ]
    reserva = models.OneToOneField(Reserva, on_delete=models.CASCADE, related_name='pago')
    metodo = models.CharField(max_length=20, choices=METODOS)
    monto = models.DecimalField(max_digits=10, decimal_places=2)
    estado = models.CharField(max_length=20, choices=ESTADOS, default='pendiente')
```
- `OneToOneField` con Reserva (una reserva = un pago)
- Múltiples métodos de pago
- Estados para tracking del pago

**Áreas de mejora importantes:**

1. **Falta validación de capacidad del evento**:

```python
from django.core.exceptions import ValidationError

class Reserva(models.Model):
    # ... campos existentes ...
    
    def clean(self):
        # Validar que no se exceda la capacidad
        reservas_confirmadas = Reserva.objects.filter(
            evento=self.evento,
            estado='confirmada'
        ).aggregate(total=Sum('cantidad'))['total'] or 0
        
        if reservas_confirmadas + self.cantidad > self.evento.capacidad:
            raise ValidationError('No hay suficiente capacidad para esta reserva')
    
    def save(self, *args, **kwargs):
        self.full_clean()
        super().save(*args, **kwargs)
```

2. **Falta validación de disponibilidad de tickets**:

```python
class Reserva(models.Model):
    def clean(self):
        # Validar disponibilidad del tipo de ticket
        if self.cantidad > self.tipo_ticket.cantidad_disponible:
            raise ValidationError(
                f'Solo hay {self.tipo_ticket.cantidad_disponible} tickets disponibles'
            )
```

3. **Falta método para generar código único de ticket**:

```python
import uuid

class Ticket(models.Model):
    # ... campos existentes ...
    
    def save(self, *args, **kwargs):
        if not self.codigo:
            # Generar código único
            self.codigo = f"TICKET-{uuid.uuid4().hex[:8].upper()}"
        super().save(*args, **kwargs)
```

4. **Falta validación de fecha futura para eventos**:

```python
from django.utils import timezone

class Evento(models.Model):
    # ... campos existentes ...
    
    def clean(self):
        if self.fecha < timezone.now().date():
            raise ValidationError('La fecha del evento debe ser futura')
```

5. **Agregar método para reducir disponibilidad al confirmar reserva**:

```python
from django.db import transaction

class Reserva(models.Model):
    @transaction.atomic
    def confirmar(self):
        if self.estado != 'pendiente':
            raise ValueError('Solo se pueden confirmar reservas pendientes')
        
        # Reducir cantidad disponible
        self.tipo_ticket.cantidad_disponible -= self.cantidad
        self.tipo_ticket.save(update_fields=['cantidad_disponible'])
        
        # Generar tickets individuales
        for i in range(self.cantidad):
            Ticket.objects.create(
                reserva=self,
                precio_final=self.tipo_ticket.precio
            )
        
        # Cambiar estado
        self.estado = 'confirmada'
        self.save(update_fields=['estado'])
```

---

### Vistas y Lógica de Negocio

**Archivo:** `apps/eventos/views.py`

Implementaron vistas basadas en clases de Django (Class-Based Views).

**EventoListView - Análisis:**

```python
class EventoListView(ListView):
    model = Evento
    template_name = 'eventos/catalogo_eventos.html'
    context_object_name = 'eventos'
    ordering = ['fecha']
    paginate_by = 10
    
    def get_queryset(self):
        qs = (
            super()
            .get_queryset()
            .select_related('categoria', 'lugar')  # Optimización!
        )
```

**Aspectos positivos:**

1. **Optimización con select_related**: Evita queries N+1 al cargar categoría y lugar.

2. **Filtros implementados**:
   - Por categoría
   - Por rango de fechas
   - Búsqueda por nombre

3. **Paginación**: 10 eventos por página.

4. **Preservación de filtros en paginación**:
```python
query_params = self.request.GET.copy()
query_params.pop('page', None)
context['pagination_query'] = query_params.urlencode()
```

**Mejoras sugeridas:**

1. **Agregar filtro por ciudad**:

```python
def get_queryset(self):
    qs = super().get_queryset().select_related('categoria', 'lugar')
    
    # Filtro por ciudad
    ciudad = self.request.GET.get('ciudad')
    if ciudad:
        qs = qs.filter(lugar__ciudad__icontains=ciudad)
    
    return qs
```

2. **Filtrar solo eventos futuros por defecto**:

```python
from django.utils import timezone

def get_queryset(self):
    qs = super().get_queryset().select_related('categoria', 'lugar')
    
    # Solo eventos futuros
    qs = qs.filter(fecha__gte=timezone.now().date())
    
    return qs
```

3. **Agregar ordenamiento por popularidad**:

```python
from django.db.models import Count

def get_queryset(self):
    qs = super().get_queryset().select_related('categoria', 'lugar')
    
    orden = self.request.GET.get('orden', 'fecha')
    if orden == 'popularidad':
        qs = qs.annotate(
            num_reservas=Count('reservas')
        ).order_by('-num_reservas')
    else:
        qs = qs.order_by('fecha')
    
    return qs
```

---

### Containerización con Docker

**Archivo:** `docker-compose.yml`

Configuración correcta con PostgreSQL, templates y healthcheck.

**Recomendación:** Agregar servicio de Redis para caché:

```yaml
services:
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
  
  web:
    depends_on:
      - db
      - redis
    environment:
      - REDIS_URL=redis://redis:6379/0
```

---

### Análisis Detallado de Arquitectura

**Lo que funciona bien:**
- Separación en apps (`eventos`, `pagos`, `reservas`) — dominio claro
- Uso de Django Signals para automatizar generación de tickets post-pago
- `bulk_create` para tickets — buena práctica de performance
- `select_related('evento', 'tipo_ticket')` en queries de reservas
- Campos de coordenadas en `Lugar` para geolocalización

**Problemas arquitectónicos:**

1. **Race condition en `crear_reserva` — descuento de stock sin transacción atómica:**

```python
# views.py actual
reserva = form.save(commit=False)
reserva.save()
tipo = reserva.tipo_ticket
tipo.cantidad_disponible -= reserva.cantidad  # ← no atómico
tipo.save()
```

Dos usuarios pueden reservar simultáneamente y sobrepasar el stock. Solución:

```python
from django.db import transaction
from django.db.models import F

@transaction.atomic
def crear_reserva(request, evento_id):
    tipo = TipoTicket.objects.select_for_update().get(pk=tipo_id)
    if tipo.cantidad_disponible < cantidad:
        raise ValueError("Stock insuficiente")
    TipoTicket.objects.filter(pk=tipo.pk).update(
        cantidad_disponible=F('cantidad_disponible') - cantidad
    )
```

2. **Signal de tickets sin protección de idempotencia:**

```python
@receiver(post_save, sender=Pago)
def confirmar_reserva_y_generar_tickets(sender, instance, ...):
    if not reserva.tickets.exists():      # ← race condition window
        Ticket.objects.bulk_create(tickets)
```

Si el signal se ejecuta dos veces (reintento, Celery), se crean tickets duplicados. Envolver en `@transaction.atomic` + `select_for_update`.

3. **`_codigo_unico()` trunca UUID a 20 caracteres** — reduce dramáticamente la unicidad. Con `bulk_create` no hay validación de unique en el batch. Usar UUID completo o agregar `unique=True` en el campo.

4. **`Evento` tiene `fecha` y `hora` separados** — complicación innecesaria. Usar `DateTimeField` unifica la representación y simplifica comparaciones.

5. **Sin tests.** No hay ningún test para reservas, pagos o generación de tickets.


---

## Requisitos para Entrega 2 (Rúbrica)

### 1. Correcciones de la Parte 1 (10%)

- [ ] Agregar descuento de `cantidad_disponible` al crear reservas
- [ ] Unificar `fecha` y `hora` en `DateTimeField` en modelo `Evento`
- [ ] Envolver signal de tickets en `@transaction.atomic` + `select_for_update`
- [ ] Garantizar unicidad de códigos de ticket (usar UUID completo o `unique=True`)
- [ ] Mover lógica de signal a un servicio explícito

### 2. Diagrama de arquitectura actualizado (5%)

Entregar diagrama que refleje la arquitectura actual del sistema: capas (presentación, servicios, dominio, persistencia), componentes principales, y las interfaces abstractas (DIP). Formato legible (draw.io, Mermaid, PlantUML).

### 3. Servicios implementados (30%)

Servicio de reservas (con control de stock atómico), servicio de pagos, servicio de generación de tickets

Cada servicio debe:
- Estar en un archivo `services.py` dentro de su app
- Encapsular la lógica de negocio (las vistas solo delegan)
- Usar `@transaction.atomic` donde haya operaciones de escritura múltiples

### 4. Inversión de dependencias (15%)

Crear una interfaz abstracta para generación de tickets:

```python
# tickets/base.py
from abc import ABC, abstractmethod

class TicketGenerator(ABC):
    @abstractmethod
    def generate(self, reserva) -> list: ...

# tickets/qr_generator.py
class QRTicketGenerator(TicketGenerator):
    def generate(self, reserva) -> list:
        # Generar tickets con código QR
        ...

# tickets/pdf_generator.py
class PDFTicketGenerator(TicketGenerator):
    def generate(self, reserva) -> list:
        # Generar tickets como PDF descargable
        ...
```

### 5. Pruebas unitarias (10%)

Implementar al menos **dos pruebas unitarias** que verifiquen lógica de negocio:

```python
def test_reserva_descuenta_cantidad_disponible(self):
    # TipoTicket con cantidad_disponible=100, reservar 5 → cantidad_disponible=95

def test_no_se_puede_reservar_mas_de_lo_disponible(self):
    # TipoTicket con cantidad_disponible=2, reservar 5 → error
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
