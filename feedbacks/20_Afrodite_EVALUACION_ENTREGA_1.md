# Evaluación Técnica - Afrodite

---

## Análisis del Código Implementado

El proyecto Afrodite es una distribuidora de productos de belleza (skincare y maquillaje) implementada con Django y PostgreSQL desplegada en Docker. El equipo (David Ballesteros Jaimes, Helen Sanabria, Paulina Velásquez, Viviana Arango) entregó 40 commits bien distribuidos, lo que evidencia un proceso de desarrollo colaborativo y con trazabilidad adecuada. La arquitectura sigue el patrón MVT de Django con cuatro aplicaciones bien delimitadas: `catalogo`, `carrito`, `usuarios` y la configuración principal `Afrodite`. En general, el proyecto tiene una base sólida con modelos bien pensados y una infraestructura Docker funcional; los problemas identificados son corregibles y corresponden principalmente a inconsistencias de diseño y ausencia de prácticas de producción.

---

### Modelos de Datos

**Archivos revisados:**
- `catalogo/models.py` (Paulina Velásquez)
- `carrito/models.py` (Viviana Arango)
- `usuarios/models.py` (Helen Sanabria)

El diseño de modelos refleja un buen entendimiento del dominio. Cada archivo tiene comentario de autoría, lo que facilita la revisión y evidencia la distribución del trabajo.

**Fortalezas identificadas:**

1. **Modelo `Producto` con filtrado por tipo de piel**: La inclusión del campo `tipo_piel` en `Producto` con choices bien definidos (`todos`, `normal`, `seca`, `grasa`, `mixta`, `sensible`) permite recomendaciones personalizadas. Esta decisión de diseño es astuta porque el valor `'todos'` permite un filtrado elegante sin lógica especial en la consulta:

```python
class Producto(models.Model):
    CATEGORIAS = [('skincare', 'Skincare'), ('maquillaje', 'Maquillaje')]
    TIPO_PIEL_PRODUCTO = [
        ('todos', 'Todos'), ('normal', 'Normal'), ('seca', 'Seca'),
        ('grasa', 'Grasa'), ('mixta', 'Mixta'), ('sensible', 'Sensible')
    ]
    nombre    = models.CharField(max_length=200)
    precio    = models.DecimalField(max_digits=10, decimal_places=0)
    categoria = models.CharField(max_length=20, choices=CATEGORIAS)
    imagen    = models.ImageField(upload_to='')
    tipo_piel = models.CharField(max_length=20, choices=TIPO_PIEL_PRODUCTO, default='todos')
```

2. **Modelo de carrito completo con ciclo de vida de orden**: `carrito/models.py` modela todo el flujo transaccional: `Cart` → `CartItem` → `Order` → `Payment`. Los choices de estado en `Order` (`pending/paid/cancelled`) y en `Payment` (`pending/completed/failed`) dan una base sólida para manejar el ciclo de vida de una compra.

3. **Perfil de usuario y direcciones separados**: La separación entre `PerfilUsuario` (datos personales + tipo de piel) y `DireccionUsuario` (direcciones de envío con soporte multi-dirección y dirección predeterminada) es una decisión de diseño correcta y extensible.

**Áreas de mejora:**

1. **`Producto.precio` con `decimal_places=0` pierde precisión de centavos**: Para una tienda de productos de belleza en Colombia o Latinoamérica, los precios típicamente no tienen centavos relevantes. Sin embargo, si el negocio escala o maneja múltiples monedas, perder esa precisión es un problema. La corrección recomendada es:

```python
# Actual — pierde centavos
precio = models.DecimalField(max_digits=10, decimal_places=0)

# Corregido — mantiene 2 decimales
precio = models.DecimalField(max_digits=10, decimal_places=2)
```

2. **`CartItem.product_name` y `unit_price` son campos redundantes y nullable**: El modelo ya tiene una FK a `Producto`, por lo que `product_name` y `unit_price` como campos nullable duplican información. El propósito parece ser guardar una "foto" del producto al momento de la compra (para que si el precio cambia, el historial no se altere), pero esto se resuelve mejor con una snapshot explícita solo al confirmar la orden, no en el carrito activo:

```python
# Actual — redundante en CartItem
product_name = models.CharField(max_length=200, null=True, blank=True)
unit_price   = models.DecimalField(max_digits=10, decimal_places=2, null=True, blank=True)

# Mejor enfoque: guardar snapshot solo en OrderItem, no en CartItem
# CartItem solo necesita: cart, producto, quantity
```

---

### Vistas y Lógica de Negocio

**Archivos revisados:**
- `catalogo/views.py`
- `carrito/views.py` (Viviana Arango)

**Fortalezas identificadas:**

1. **Recomendaciones por tipo de piel en el catálogo**: La función `get_tipo_piel_usuario(request)` extrae el tipo de piel del perfil del usuario autenticado para personalizar el catálogo. El uso de `tipo_piel__in=[tipo_piel, 'todos']` es elegante y evita duplicar productos en la consulta:

```python
# Inteligente: incluye productos para el tipo de piel del usuario Y los de 'todos'
Producto.objects.filter(tipo_piel__in=[tipo_piel, 'todos'])
```

2. **Carrito DB-based completamente protegido**: Todas las vistas de `carrito/views.py` están decoradas con `@login_required`, lo cual es la práctica correcta para un sistema de carrito que persiste en base de datos.

**Áreas de mejora:**

1. **Dos implementaciones de carrito coexistiendo — inconsistencia arquitectónica**: El proyecto tiene un carrito basado en sesión en `catalogo/views.py` y otro basado en base de datos en `carrito/views.py`. Esto genera confusión sobre cuál es el sistema activo y puede causar comportamientos inesperados si ambos se usan simultáneamente. Se debe eliminar la implementación de sesión y centralizar todo en `carrito/views.py`.

2. **Lógica de checkout en la vista**: La creación de `Order` y `Payment` directamente en la vista de checkout es el problema más importante de la arquitectura actual. Esta lógica de negocio debe extraerse a una capa de servicios:

```python
# Actual — lógica de negocio mezclada en la vista
def checkout(request):
    cart = Cart.objects.get(...)
    order = Order.objects.create(user=request.user, total=...)
    payment = Payment.objects.create(order=order, method=..., status='pending')
    # ... más lógica directamente aquí

# Correcto — la vista delega a un servicio
# carrito/services.py
def procesar_checkout(user, cart, metodo_pago):
    order = Order.objects.create(user=user, total=calcular_total(cart))
    payment = Payment.objects.create(order=order, method=metodo_pago, status='pending')
    cart.is_active = False
    cart.save()
    return order

# carrito/views.py
def checkout(request):
    order = procesar_checkout(request.user, cart, metodo_pago)
    return redirect('orden_confirmada', pk=order.pk)
```

---

### Infraestructura y Despliegue

**Archivos revisados:**
- `Dockerfile`
- `docker-compose.yml`
- `README.md`
- `.gitignore` (ausencias notables)

**Fortalezas identificadas:**

1. **Dockerfile multi-stage con Python 3.12-slim**: El uso de `python:3.12-slim` reduce el tamaño de la imagen. La integración de `gunicorn` y un `entrypoint.sh` refleja una configuración pensada para producción, no solo para desarrollo.

2. **`docker-compose.yml` con tres servicios bien configurados**: La separación entre `afrodite_db` (postgres:16 con healthcheck), `afrodite` (web) y `adminer` (inspección de BD) es práctica. El healthcheck en la base de datos garantiza que el contenedor web no arranque antes de que PostgreSQL esté listo.

3. **README.md con instrucciones detalladas y guía de troubleshooting**: La documentación incluye pasos claros para levantar el proyecto y sección de resolución de problemas comunes, lo que facilita la evaluación y el onboarding de nuevos colaboradores.

**Áreas de mejora:**

1. **`db.sqlite3` committeado al repositorio**: El archivo `db.sqlite3` no debe existir en el repositorio. Debe agregarse al `.gitignore` y eliminarse del historial con `git rm --cached db.sqlite3`. Una base de datos local no tiene lugar en el control de versiones.

2. **Archivos estáticos del admin de Django committeados**: Los archivos en `staticfiles/admin/` son generados por `collectstatic` y no deben estar en el repo. Agregar `staticfiles/` al `.gitignore` y ejecutar `collectstatic` como parte del proceso de despliegue (en `entrypoint.sh` o en el `Dockerfile`).

---

### Patrones y Principios SOLID

**Lo que funciona bien:**
- Separación de responsabilidades entre aplicaciones Django (`catalogo`, `carrito`, `usuarios`) sigue el principio de responsabilidad única a nivel de módulo.
- El modelo de dominio está bien normalizado: `PerfilUsuario` y `DireccionUsuario` separados permiten múltiples direcciones por usuario sin duplicar datos de perfil.
- El uso de `choices` en campos de estado y categoría centraliza las opciones válidas en el modelo, evitando magic strings dispersos.
- La protección de vistas con `@login_required` aplica el principio de mínimo privilegio de forma consistente.

**Problemas arquitectónicos:**

1. **Ausencia de capa de servicios (violación de SRP en vistas)**: Las vistas actualmente cumplen tres roles: recibir la petición HTTP, ejecutar lógica de negocio y retornar respuesta. Esto viola el Principio de Responsabilidad Única. La lógica de `checkout` (crear orden, crear pago, desactivar carrito) debe vivir en `carrito/services.py`.

2. **Duplicación del sistema de carrito**: Mantener dos implementaciones paralelas (sesión y BD) viola el principio DRY (Don't Repeat Yourself) y crea un vector de bugs. Debe existir una única fuente de verdad para el carrito.

3. **Sin pruebas unitarias**: El proyecto no tiene tests. Para la Entrega 2, se requiere cobertura básica de los modelos y la lógica de servicios.

---

## Requisitos para Entrega 2 (Rúbrica)

### 1. Correcciones de la Parte 1 (10%)

- [ ] Eliminar `db.sqlite3` del repositorio y agregarlo al `.gitignore`
- [ ] Eliminar `staticfiles/admin/` del repositorio y agregarlo al `.gitignore`
- [ ] Eliminar la implementación de carrito por sesión en `catalogo/views.py` y usar únicamente el carrito de `carrito/views.py`
- [ ] Eliminar los campos `product_name` y `unit_price` de `CartItem` o documentar explícitamente su propósito como snapshot inmutable
- [ ] Corregir `Producto.precio` a `decimal_places=2`

### 2. Diagrama de arquitectura actualizado (5%)

- [ ] Diagrama que muestre las 3 capas: vistas → servicios → modelos/repositorio
- [ ] Incluir el flujo completo: catálogo → carrito → checkout → orden → pago
- [ ] Mostrar la relación entre `PerfilUsuario` y el sistema de recomendaciones por tipo de piel

### 3. Servicios implementados (30%)

- [ ] Crear `carrito/services.py` con función `procesar_checkout(user, cart, metodo_pago) → Order`
- [ ] Crear `catalogo/services.py` con función `obtener_productos_recomendados(usuario) → QuerySet`
- [ ] Mover la lógica de negocio actualmente en vistas hacia los servicios correspondientes
- [ ] Las vistas deben limitarse a recibir la petición y delegar al servicio

### 4. Inversión de dependencias (15%)

- [ ] Los servicios deben recibir dependencias por parámetro, no instanciarlas internamente
- [ ] Definir interfaces o abstracciones para los servicios si se usan en múltiples vistas
- [ ] Ejemplo: `procesar_checkout` no debe crear el carrito internamente, debe recibirlo como parámetro

### 5. Pruebas unitarias (10%)

- [ ] Tests para `Producto` — verificar filtrado por tipo de piel
- [ ] Tests para el servicio de checkout — verificar que se crea `Order` y `Payment` correctamente
- [ ] Tests para el servicio de recomendaciones — verificar que retorna productos del tipo correcto y los de `'todos'`
- [ ] Al menos 6 pruebas unitarias en total usando `django.test.TestCase`

### 6. Calidad del código y arquitectura (15%)

- [ ] Sin lógica de negocio en vistas (solo llamadas a servicios)
- [ ] Un único sistema de carrito (eliminar implementación por sesión)
- [ ] Variables y funciones con nombres en español consistentes con el dominio del negocio
- [ ] Sin archivos generados (SQLite, staticfiles) en el repositorio

### 7. Despliegue en nube + sistema de dos idiomas (15%)

- [ ] URL pública funcional del proyecto desplegado
- [ ] Configuración de idiomas (español / inglés) usando `django.utils.translation`
- [ ] Archivos de traducción `.po` / `.mo` generados con `makemessages` / `compilemessages`
- [ ] `LANGUAGE_CODE`, `USE_I18N = True` y `LocaleMiddleware` configurados en `settings.py`
