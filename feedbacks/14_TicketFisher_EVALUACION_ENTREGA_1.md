# Evaluación Técnica - TicketFisher

## Nota Final: 4.49/5.0

---

## Análisis del Código Implementado

He revisado el código de su plataforma de venta de tickets para eventos. El proyecto demuestra una implementación sólida con Django tradicional (templates), sistema de roles personalizado y arquitectura modular.

### Modelos de Datos

**Archivos revisados:**
- `apps/accounts/models.py` (55 líneas, 2 modelos)
- `apps/events/models.py` (23 líneas, 1 modelo)
- `apps/access/models.py` (vacío)

Su modelado de datos es funcional con un sistema de roles bien implementado.

**Modelo Usuario - Personalización de AbstractUser:**

```python
class Usuario(AbstractUser):
    """Modelo personalizado de usuario"""
    rol = models.ForeignKey(Rol, on_delete=models.SET_NULL, null=True, blank=True, related_name='usuarios')
    telefono = models.CharField(max_length=20, blank=True)
    fecha_nacimiento = models.DateField(null=True, blank=True)
    cedula = models.CharField(max_length=20, unique=True, blank=True, null=True)
    genero = models.CharField(
        max_length=10,
        choices=[('M', 'Masculino'), ('F', 'Femenino'), ('O', 'Otro')],
        blank=True
    )
    activo = models.BooleanField(default=True)
    verificado = models.BooleanField(default=False)
    foto_perfil = models.ImageField(upload_to='perfil/', null=True, blank=True)
    ultimo_acceso = models.DateTimeField(null=True, blank=True)
    
    def es_admin(self):
        return self.rol and self.rol.nombre == 'admin'
    
    def es_organizador(self):
        return self.rol and self.rol.nombre == 'organizador'
    
    def es_usuario(self):
        return self.rol and self.rol.nombre == 'usuario'
```

**Aspectos destacados:**

1. **Herencia de AbstractUser**: Excelente práctica para extender el modelo de usuario de Django.

2. **Sistema de roles flexible**: Relación ForeignKey con modelo Rol permite roles dinámicos.

3. **Métodos helper**: `es_admin()`, `es_organizador()`, `es_usuario()` facilitan verificación de permisos.

4. **Campos de perfil completos**: Teléfono, cédula, género, foto de perfil.

5. **Flags de estado**: `activo` y `verificado` para control de acceso.

**Modelo Rol - Sistema de Permisos:**

```python
class Rol(models.Model):
    """Modelo para definir roles del sistema"""
    nombre = models.CharField(max_length=100, unique=True)
    descripcion = models.TextField(blank=True)
    permisos = models.JSONField(default=dict, blank=True)
    activo = models.BooleanField(default=True)
    fecha_creacion = models.DateTimeField(auto_now_add=True)
```

**Excelente diseño:**
- **JSONField para permisos**: Flexible para almacenar permisos personalizados
- **nombre unique**: Previene duplicados
- **activo**: Permite desactivar roles sin eliminarlos

**Modelo Evento - Básico pero Funcional:**

```python
class Evento(models.Model):
    """Modelo para eventos"""
    nombre = models.CharField(max_length=200)
    lugar = models.CharField(max_length=255)
    fecha = models.DateField()
    hora = models.TimeField()
    descripcion = models.TextField(blank=True)
    organizador = models.ForeignKey(Usuario, on_delete=models.CASCADE, related_name='eventos')
    activo = models.BooleanField(default=True)
```

**Áreas de mejora importantes:**

1. **Falta modelo de Ticket/Reserva**:

```python
class TipoTicket(models.Model):
    evento = models.ForeignKey(Evento, on_delete=models.CASCADE, related_name='tipos_ticket')
    nombre = models.CharField(max_length=100)
    precio = models.DecimalField(max_digits=10, decimal_places=2)
    cantidad_disponible = models.PositiveIntegerField()
    descripcion = models.TextField(blank=True)

class Reserva(models.Model):
    ESTADOS = [
        ('pendiente', 'Pendiente'),
        ('confirmada', 'Confirmada'),
        ('cancelada', 'Cancelada'),
    ]
    
    usuario = models.ForeignKey(Usuario, on_delete=models.CASCADE, related_name='reservas')
    evento = models.ForeignKey(Evento, on_delete=models.CASCADE, related_name='reservas')
    tipo_ticket = models.ForeignKey(TipoTicket, on_delete=models.CASCADE)
    cantidad = models.PositiveIntegerField()
    estado = models.CharField(max_length=20, choices=ESTADOS, default='pendiente')
    fecha_reserva = models.DateTimeField(auto_now_add=True)

class Ticket(models.Model):
    reserva = models.ForeignKey(Reserva, on_delete=models.CASCADE, related_name='tickets')
    codigo = models.CharField(max_length=100, unique=True)
    precio_final = models.DecimalField(max_digits=10, decimal_places=2)
    usado = models.BooleanField(default=False)
    fecha_uso = models.DateTimeField(null=True, blank=True)
```

2. **Agregar validaciones al modelo Evento**:

```python
from django.core.exceptions import ValidationError
from django.utils import timezone

class Evento(models.Model):
    # ... campos existentes ...
    capacidad = models.PositiveIntegerField(default=0)
    
    def clean(self):
        if self.fecha < timezone.now().date():
            raise ValidationError('La fecha del evento debe ser futura')
    
    @property
    def tickets_vendidos(self):
        return Reserva.objects.filter(
            evento=self,
            estado='confirmada'
        ).aggregate(total=Sum('cantidad'))['total'] or 0
    
    @property
    def tickets_disponibles(self):
        return self.capacidad - self.tickets_vendidos
```

3. **Implementar sistema de permisos basado en JSONField**:

```python
class Rol(models.Model):
    # ... campos existentes ...
    
    def tiene_permiso(self, permiso):
        """Verifica si el rol tiene un permiso específico"""
        return self.permisos.get(permiso, False)
    
    def agregar_permiso(self, permiso):
        """Agrega un permiso al rol"""
        self.permisos[permiso] = True
        self.save(update_fields=['permisos'])

# Uso en vistas
def crear_evento(request):
    if not request.user.rol.tiene_permiso('crear_eventos'):
        return HttpResponseForbidden('No tienes permiso para crear eventos')
```

4. **Agregar modelo de Pago**:

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
    referencia = models.CharField(max_length=100, blank=True)
    fecha_pago = models.DateTimeField(auto_now_add=True)
```

---

### Arquitectura del Proyecto

**Separación de apps:**
- `accounts`: Usuarios y roles
- `events`: Eventos
- `access`: Control de acceso (vacío actualmente)
- `panel_admin`: Panel administrativo
- `organizer`: Panel de organizadores
- `end_user`: Panel de usuarios finales

Esta separación es buena y facilita el mantenimiento.

---

### Containerización con Docker

**Archivo:** `docker-compose.yml`

Configuración correcta con PostgreSQL y templates.

---

### Buenas Prácticas de Git y Control de Versiones

**Análisis del historial de commits:** 11 commits totales

**Evaluación: Buenas prácticas de Git**

**Aspectos positivos:**

- **Historial moderado**: 11 commits es razonable para el alcance actual.
- **Mensajes descriptivos**: 90% de los commits tienen mensajes claros.
- **Commits rastreables**: Los cambios están documentados.

**Área de mejora:**

- **Múltiples autores (4 diferentes)**: Asegúrense de configurar Git correctamente.

**Recomendaciones:**

1. **Adoptar Conventional Commits**:
   - `feat:` nuevas funcionalidades
   - `fix:` correcciones
   - `docs:` documentación

2. **Commits más frecuentes**: Hacer commits después de cada funcionalidad completa.

---

## Recomendaciones Específicas para la Segunda Entrega

### 1. Completar Sistema de Tickets y Reservas

Implementar los modelos faltantes (TipoTicket, Reserva, Ticket, Pago) como se mostró arriba.

### 2. Sistema de Validación de Tickets con QR

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

# Vista para validar entrada
def validar_ticket(request, codigo):
    try:
        ticket = Ticket.objects.get(codigo=codigo)
        
        if ticket.usado:
            return JsonResponse({'valido': False, 'mensaje': 'Ticket ya usado'})
        
        ticket.usado = True
        ticket.fecha_uso = timezone.now()
        ticket.save()
        
        return JsonResponse({
            'valido': True,
            'evento': ticket.reserva.evento.nombre,
            'usuario': ticket.reserva.usuario.get_full_name()
        })
    except Ticket.DoesNotExist:
        return JsonResponse({'valido': False, 'mensaje': 'Ticket no encontrado'})
```

### 3. Dashboard de Organizador

```python
from django.db.models import Sum, Count

def dashboard_organizador(request):
    eventos = Evento.objects.filter(organizador=request.user)
    
    estadisticas = []
    for evento in eventos:
        stats = {
            'evento': evento,
            'tickets_vendidos': Reserva.objects.filter(
                evento=evento,
                estado='confirmada'
            ).aggregate(total=Sum('cantidad'))['total'] or 0,
            'ingresos': Pago.objects.filter(
                reserva__evento=evento,
                estado='aprobado'
            ).aggregate(total=Sum('monto'))['total'] or 0,
            'asistentes_confirmados': Ticket.objects.filter(
                reserva__evento=evento,
                usado=True
            ).count()
        }
        estadisticas.append(stats)
    
    return render(request, 'organizer/dashboard.html', {
        'estadisticas': estadisticas
    })
```

### 4. Sistema de Notificaciones

```python
from django.core.mail import send_mail

def enviar_confirmacion_reserva(reserva):
    send_mail(
        subject=f'Confirmación de reserva - {reserva.evento.nombre}',
        message=f'Tu reserva para {reserva.evento.nombre} ha sido confirmada',
        from_email='noreply@ticketfisher.com',
        recipient_list=[reserva.usuario.email],
    )

def enviar_recordatorio_evento(evento):
    """Enviar recordatorio 24 horas antes"""
    reservas = Reserva.objects.filter(
        evento=evento,
        estado='confirmada'
    ).select_related('usuario')
    
    for reserva in reservas:
        send_mail(
            subject=f'Recordatorio: {evento.nombre} es mañana',
            message=f'Tu evento {evento.nombre} es mañana a las {evento.hora}',
            from_email='noreply@ticketfisher.com',
            recipient_list=[reserva.usuario.email],
        )
```

### 5. Decoradores de Permisos Personalizados

```python
from functools import wraps
from django.http import HttpResponseForbidden

def requiere_permiso(permiso):
    def decorator(view_func):
        @wraps(view_func)
        def wrapper(request, *args, **kwargs):
            if not request.user.is_authenticated:
                return redirect('login')
            
            if not request.user.rol or not request.user.rol.tiene_permiso(permiso):
                return HttpResponseForbidden('No tienes permiso para esta acción')
            
            return view_func(request, *args, **kwargs)
        return wrapper
    return decorator

# Uso
@requiere_permiso('crear_eventos')
def crear_evento(request):
    # ...
```

### 6. Sistema de Reseñas de Eventos

```python
class ReseñaEvento(models.Model):
    evento = models.ForeignKey(Evento, on_delete=models.CASCADE, related_name='reseñas')
    usuario = models.ForeignKey(Usuario, on_delete=models.CASCADE)
    calificacion = models.PositiveSmallIntegerField(
        validators=[MinValueValidator(1), MaxValueValidator(5)]
    )
    comentario = models.TextField(blank=True)
    fecha = models.DateTimeField(auto_now_add=True)
    
    class Meta:
        unique_together = ('evento', 'usuario')
    
    def save(self, *args, **kwargs):
        # Solo permitir reseñas de usuarios que asistieron
        asistio = Ticket.objects.filter(
            reserva__usuario=self.usuario,
            reserva__evento=self.evento,
            usado=True
        ).exists()
        
        if not asistio:
            raise ValidationError('Solo puedes reseñar eventos a los que asististe')
        
        super().save(*args, **kwargs)
```

---

## Aspectos Positivos del Proyecto

- **Sistema de roles personalizado** con JSONField para permisos
- **Usuario extendido** con AbstractUser (buena práctica)
- **Métodos helper** para verificación de roles
- **Separación de apps** por tipo de usuario (admin, organizador, end_user)
- **Templates organizados** por app
- **Docker** configurado correctamente
- **11 commits** demuestran desarrollo continuo

---

## Próximos Pasos

1. **Completar modelos** de Ticket, Reserva, Pago
2. **Implementar QR codes** para validación de entrada
3. **Dashboard de organizador** con estadísticas
4. **Sistema de notificaciones** por email
5. **Decoradores de permisos** personalizados
6. **Sistema de reseñas** para eventos
7. **Tests** para sistema de permisos
8. **Documentar** sistema de roles en README

Su proyecto tiene una base sólida con un sistema de roles bien diseñado. Para la segunda entrega, enfóquense en completar los modelos de ticketing, implementar QR codes para validación y preparar una demo que muestre el flujo completo desde la compra hasta la entrada al evento.

---

*Evaluación realizada sobre el código fuente del repositorio GitHub.*
