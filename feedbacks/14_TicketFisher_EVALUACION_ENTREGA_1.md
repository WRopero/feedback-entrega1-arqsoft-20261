# Evaluación Técnica - TicketFisher

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
