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

### Buenas Prácticas de Git y Control de Versiones

**Análisis del historial de commits:** 13 commits totales

**Evaluación: Buenas prácticas de Git**

**Aspectos positivos:**

- **Historial moderado**: 13 commits es razonable para el alcance del proyecto.
- **Mensajes descriptivos**: 100% de los commits tienen mensajes claros.
- **Commits rastreables**: Los cambios están bien documentados.

**Área de mejora:**

- **Múltiples autores (4 diferentes)**: Asegúrense de configurar Git correctamente en todos los equipos.

**Recomendaciones:**

1. **Adoptar Conventional Commits**:
   - `feat:` nuevas funcionalidades
   - `fix:` correcciones
   - `docs:` documentación

2. **Commits más frecuentes**: Hacer commits más pequeños y atómicos.

---

## Recomendaciones Específicas para la Segunda Entrega

### 1. Implementar Sistema de QR para Tickets

```python
import qrcode
from io import BytesIO
from django.core.files import File

class Ticket(models.Model):
    # ... campos existentes ...
    qr_code = models.ImageField(upload_to='qr_codes/', blank=True, null=True)
    
    def generate_qr_code(self):
        qr = qrcode.QRCode(version=1, box_size=10, border=5)
        qr.add_data(self.codigo)
        qr.make(fit=True)
        
        img = qr.make_image(fill_color="black", back_color="white")
        buffer = BytesIO()
        img.save(buffer, format='PNG')
        
        self.qr_code.save(f'qr_{self.codigo}.png', File(buffer), save=False)
        self.save()
```

### 2. Agregar Sistema de Notificaciones por Email

```python
from django.core.mail import send_mail
from django.template.loader import render_to_string

def enviar_confirmacion_reserva(reserva):
    context = {
        'reserva': reserva,
        'evento': reserva.evento,
        'tickets': reserva.tickets.all(),
    }
    
    html_message = render_to_string('emails/confirmacion_reserva.html', context)
    
    send_mail(
        subject=f'Confirmación de reserva - {reserva.evento.nombre}',
        message='',
        html_message=html_message,
        from_email='noreply@vibepass.com',
        recipient_list=[reserva.usuario.email],
    )
```

### 3. Implementar Sistema de Recordatorios

```python
from django.utils import timezone
from datetime import timedelta

def enviar_recordatorios_eventos():
    """Enviar recordatorios 24 horas antes del evento"""
    manana = timezone.now().date() + timedelta(days=1)
    
    eventos_manana = Evento.objects.filter(fecha=manana)
    
    for evento in eventos_manana:
        reservas = Reserva.objects.filter(
            evento=evento,
            estado='confirmada'
        ).select_related('usuario')
        
        for reserva in reservas:
            send_mail(
                subject=f'Recordatorio: {evento.nombre} es mañana',
                message=f'Tu evento {evento.nombre} es mañana a las {evento.hora}',
                from_email='noreply@vibepass.com',
                recipient_list=[reserva.usuario.email],
            )
```

### 4. Agregar Dashboard de Organizador

```python
from django.db.models import Sum, Count

class EventoDashboardView(DetailView):
    model = Evento
    template_name = 'eventos/dashboard.html'
    
    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        evento = self.object
        
        # Estadísticas
        context['total_reservas'] = evento.reservas.filter(
            estado='confirmada'
        ).count()
        
        context['tickets_vendidos'] = evento.reservas.filter(
            estado='confirmada'
        ).aggregate(total=Sum('cantidad'))['total'] or 0
        
        context['ingresos_totales'] = evento.reservas.filter(
            estado='confirmada',
            pago__estado='aprobado'
        ).aggregate(total=Sum('pago__monto'))['total'] or 0
        
        context['capacidad_ocupada'] = (
            context['tickets_vendidos'] / evento.capacidad * 100
        )
        
        return context
```

### 5. Implementar Sistema de Reembolsos

```python
class Reserva(models.Model):
    # ... campos existentes ...
    
    @transaction.atomic
    def cancelar_con_reembolso(self):
        if self.estado != 'confirmada':
            raise ValueError('Solo se pueden cancelar reservas confirmadas')
        
        # Verificar que falten más de 48 horas para el evento
        dias_faltantes = (self.evento.fecha - timezone.now().date()).days
        if dias_faltantes < 2:
            raise ValueError('No se pueden cancelar reservas con menos de 48 horas de anticipación')
        
        # Devolver disponibilidad
        self.tipo_ticket.cantidad_disponible += self.cantidad
        self.tipo_ticket.save()
        
        # Marcar tickets como no usados
        self.tickets.update(usado=False)
        
        # Cambiar estado
        self.estado = 'cancelada'
        self.save()
        
        # Procesar reembolso
        if hasattr(self, 'pago') and self.pago.estado == 'aprobado':
            # Aquí integrar con pasarela de pagos para reembolso
            pass
```

### 6. Agregar Validación de Entrada con QR

```python
class ValidarTicketView(View):
    def post(self, request):
        codigo = request.POST.get('codigo')
        
        try:
            ticket = Ticket.objects.select_related(
                'reserva__evento'
            ).get(codigo=codigo)
            
            # Validar que no esté usado
            if ticket.usado:
                return JsonResponse({
                    'valido': False,
                    'mensaje': 'Este ticket ya fue usado'
                })
            
            # Validar que sea para hoy
            if ticket.reserva.evento.fecha != timezone.now().date():
                return JsonResponse({
                    'valido': False,
                    'mensaje': 'Este ticket no es para hoy'
                })
            
            # Marcar como usado
            ticket.usado = True
            ticket.save()
            
            return JsonResponse({
                'valido': True,
                'mensaje': 'Ticket válido',
                'evento': ticket.reserva.evento.nombre,
                'tipo': ticket.reserva.tipo_ticket.nombre
            })
            
        except Ticket.DoesNotExist:
            return JsonResponse({
                'valido': False,
                'mensaje': 'Ticket no encontrado'
            })
```

---

## Aspectos Positivos del Proyecto

- Modelado de datos completo para sistema de ticketing
- Sistema de tipos de tickets flexible
- Optimización de queries con select_related
- Filtros y búsqueda implementados
- Paginación con preservación de filtros
- Docker configurado correctamente
- Templates organizados por app
- Sistema de pagos con múltiples métodos

---

## Próximos Pasos

1. **Agregar validaciones** de capacidad y disponibilidad
2. **Implementar QR codes** para tickets
3. **Sistema de notificaciones** por email
4. **Dashboard de organizador** con estadísticas
5. **Sistema de reembolsos** con políticas
6. **Validación de entrada** con escaneo de QR
7. **Tests** para funcionalidades críticas
8. **Documentar** flujos de negocio en README

Su proyecto tiene una base sólida con modelos bien diseñados. Para la segunda entrega, enfóquense en completar las validaciones, agregar el sistema de QR codes y preparar una demo que muestre el flujo completo desde la compra hasta la entrada al evento.

---

*Evaluación realizada sobre el código fuente del repositorio GitHub.*
