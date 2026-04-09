# Evaluación Técnica - Code Academy

## Nota Final: 3.51/5.0

---

## Análisis del Código Implementado

He revisado su plataforma de eCommerce para cursos y libros de programación. El proyecto muestra una arquitectura bien pensada con separación de apps Django y modelos complejos, aunque falta implementación de Docker completa.

### Modelos de Datos

**Archivos revisados:** 
- `app/products/models.py` (266 líneas, 7 modelos)
- `app/orders/models.py` (89 líneas, 2 modelos)

Su modelado de datos es sofisticado y demuestra comprensión avanzada de Django:

**Product**: Modelo polimórfico que maneja tanto cursos como libros usando un campo `type`. Esto es una decisión de diseño inteligente que evita duplicación de código.

```python
TYPE_COURSE = 'course'
TYPE_BOOK = 'book'
TYPE_CHOICES = [
    (TYPE_COURSE, 'Curso'),
    (TYPE_BOOK, 'Libro'),
]
```

**Fortalezas identificadas:**

1. **Validadores en campos numéricos**: Uso correcto de `MinValueValidator` y `MaxValueValidator` para precio y rating.

2. **Relaciones bien diseñadas**: 
   - `Category` → `Product` (muchos a uno con PROTECT)
   - `Product` → `Chapter` (uno a muchos con CASCADE)
   - `limit_choices_to` para restringir relaciones solo a cursos o libros

3. **Soft delete**: Campo `is_active` para ocultar productos sin eliminarlos de la base de datos.

4. **Timestamps automáticos**: `auto_now_add` y `auto_now` para auditoría.

5. **Modelos avanzados implementados**:
   - `BookDownload`: Control de descargas con límite (3 descargas máximo)
   - `CourseProgress`: Tracking de progreso con ManyToMany a capítulos completados
   - `CourseCertificate`: Generación de certificados PDF al completar cursos

6. **Método de negocio en modelo**:

```python
def recalculate_progress(self):
    total = self.product.chapters.count()
    completed = self.completed_chapters.count()
    self.progress_percentage = 0 if total == 0 else int((completed / total) * 100)
    self.save(update_fields=['progress_percentage', 'updated_at'])
```

**Áreas de mejora:**

1. **Falta validación de coherencia entre tipo y campos específicos**. Un libro no debería tener `duration` y un curso no debería tener `pages`:

```python
from django.core.exceptions import ValidationError

class Product(models.Model):
    # ... campos existentes ...
    
    def clean(self):
        if self.type == self.TYPE_COURSE and self.pages:
            raise ValidationError('Un curso no puede tener páginas')
        if self.type == self.TYPE_BOOK and self.duration:
            raise ValidationError('Un libro no puede tener duración')
        if self.type == self.TYPE_BOOK and not self.book_file:
            raise ValidationError('Un libro debe tener archivo PDF')
```

2. **El modelo Order tiene snapshot pero podría mejorar**:

```python
# Agregar campos para auditoría completa
class Order(models.Model):
    # ... campos existentes ...
    user_email = models.EmailField()  # Snapshot del email del usuario
    user_name = models.CharField(max_length=255)  # Snapshot del nombre
    ip_address = models.GenericIPAddressField(null=True)  # Para seguridad
```

3. **Falta índice compuesto para consultas frecuentes**:

```python
class Product(models.Model):
    class Meta:
        indexes = [
            models.Index(fields=['type', 'is_active', '-created_at']),
            models.Index(fields=['category', 'is_active']),
        ]
```

---

### Vistas y Lógica de Negocio

**Archivo:** `app/products/views.py` (216 líneas)

Implementaron vistas basadas en clases de DRF con filtros avanzados.

**Aspectos destacados:**

1. **Uso de DjangoFilterBackend**: Filtros por tipo, nivel, idioma, categoría.

2. **SearchFilter**: Búsqueda en título, autor y descripción.

3. **OrderingFilter**: Ordenamiento por precio, rating, fecha.

4. **Optimización de queries**:

```python
def get_queryset(self):
    return (
        Product.objects
        .filter(is_active=True)
        .select_related('category')
        .prefetch_related('chapters', 'table_of_contents')
    )
```

5. **Anotaciones para evitar queries N+1**:

```python
Category.objects
    .annotate(active_products=Count('products', filter=Q(products__is_active=True)))
    .filter(active_products__gt=0)
```

**Problemas encontrados:**

1. **Falta Docker completo**. El README menciona Docker pero no encontré `docker-compose.yml` funcional en el análisis. Esto es **crítico** para la rúbrica.

2. **No hay control de permisos en vistas de descarga**. Cualquiera podría acceder si conoce la URL:

```python
class BookDownloadView(APIView):
    permission_classes = [IsAuthenticated]  # DEBE estar autenticado
    
    def get(self, request, product_id):
        product = get_object_or_404(Product, id=product_id, type=Product.TYPE_BOOK)
        
        # Verificar que el usuario compró el libro
        has_purchased = Order.objects.filter(
            user=request.user,
            status=Order.STATUS_COMPLETED,
            items__product=product
        ).exists()
        
        if not has_purchased:
            return Response(
                {'error': 'Debes comprar este libro para descargarlo'},
                status=status.HTTP_403_FORBIDDEN
            )
        
        # Verificar límite de descargas
        download, created = BookDownload.objects.get_or_create(
            user=request.user,
            product=product
        )
        
        if download.download_count >= download.max_downloads:
            return Response(
                {'error': f'Has alcanzado el límite de {download.max_downloads} descargas'},
                status=status.HTTP_403_FORBIDDEN
            )
        
        # Incrementar contador
        download.download_count += 1
        download.last_downloaded_at = timezone.now()
        download.save()
        
        # Servir archivo
        return FileResponse(
            product.book_file.open('rb'),
            as_attachment=True,
            filename=f"{product.title}.pdf"
        )
```

3. **Integración con Stripe mencionada pero no visible en el código revisado**. Esto debería estar en `orders/views.py`.

---

### Integración con Stripe

El README menciona integración con Stripe para pagos, lo cual es excelente. Sin embargo, necesitan asegurar:

**Webhook de Stripe para actualizar estado de orden**:

```python
import stripe
from django.conf import settings
from django.views.decorators.csrf import csrf_exempt

stripe.api_key = settings.STRIPE_SECRET_KEY

@csrf_exempt
def stripe_webhook(request):
    payload = request.body
    sig_header = request.META['HTTP_STRIPE_SIGNATURE']
    
    try:
        event = stripe.Webhook.construct_event(
            payload, sig_header, settings.STRIPE_WEBHOOK_SECRET
        )
    except ValueError:
        return HttpResponse(status=400)
    except stripe.error.SignatureVerificationError:
        return HttpResponse(status=400)
    
    if event['type'] == 'payment_intent.succeeded':
        payment_intent = event['data']['object']
        
        # Actualizar orden
        order = Order.objects.get(payment_reference=payment_intent['id'])
        order.status = Order.STATUS_COMPLETED
        order.save()
        
        # Enviar email de confirmación
        send_order_confirmation_email(order)
    
    return HttpResponse(status=200)
```

---

### Arquitectura de Apps

Excelente separación en apps Django:
- `core`: Configuración base
- `users`: Autenticación y usuarios
- `products`: Catálogo de productos
- `orders`: Órdenes y pagos

Esto facilita el mantenimiento y escalabilidad.

---

### Buenas Prácticas de Git y Control de Versiones

**Análisis del historial de commits:** 29 commits totales

**Aspectos positivos (Excelente):**

- **Buen historial de commits**: 29 commits demuestra desarrollo continuo y bien versionado.
- **Mensajes descriptivos al 100%**: Todos los commits siguen convenciones claras. Ejemplos:
  - "fix(ui): restore product images and hide empty categories"
  - "feat: implement course progress tracking"
  - "refactor: optimize database queries with select_related"
- **Equipo consistente**: 3 autores trabajando coordinadamente.
- **Conventional Commits**: Ya están usando el estándar de la industria con prefijos como `feat:`, `fix:`, `refactor:`.
- **Commits rastreables**: Cambios bien documentados y organizados.

**Este es el mejor ejemplo de buenas prácticas de Git de todos los proyectos evaluados.**

**Recomendaciones para mantener la calidad:**

1. **Continuar con Conventional Commits**: Ya lo están haciendo bien, mantengan esta práctica.

2. **Considerar usar branches**: Si aún no lo hacen, usen feature branches para desarrollo:
   ```bash
   git checkout -b feature/nombre-funcionalidad
   # ... hacer cambios ...
   git commit -m "feat: descripción"
   git checkout main
   git merge feature/nombre-funcionalidad
   ```

3. **Tags para releases**: Marcar versiones importantes:
   ```bash
   git tag -a v1.0 -m "Primera entrega"
   git push origin v1.0
   ```

---

## Problema Crítico: Falta Docker

El README menciona Docker extensivamente pero **no encontré `docker-compose.yml` en el análisis del repositorio**. Esto es fundamental para la rúbrica.

**Debe crear:**

```yaml
version: '3.8'

services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: codeacademy_db
      POSTGRES_USER: codeacademy_user
      POSTGRES_PASSWORD: codeacademy_pass
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U codeacademy_user"]
      interval: 5s
      retries: 5

  web:
    build: .
    command: python manage.py runserver 0.0.0.0:8000
    volumes:
      - ./app:/app
      - media_files:/app/media
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=postgresql://codeacademy_user:codeacademy_pass@db:5432/codeacademy_db
      - STRIPE_SECRET_KEY=${STRIPE_SECRET_KEY}
      - STRIPE_PUBLISHABLE_KEY=${STRIPE_PUBLISHABLE_KEY}
    depends_on:
      db:
        condition: service_healthy

  frontend:
    build:
      context: ./frontend
    volumes:
      - ./frontend:/app
      - /app/node_modules
    ports:
      - "5173:5173"
    environment:
      - VITE_API_URL=http://web:8000
      - VITE_STRIPE_PUBLISHABLE_KEY=${VITE_STRIPE_PUBLISHABLE_KEY}
    depends_on:
      - web

volumes:
  postgres_data:
  media_files:
```

**Dockerfile para backend:**

```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 8000

CMD ["python", "manage.py", "runserver", "0.0.0.0:8000"]
```

---

## Recomendaciones Específicas para la Segunda Entrega

### 1. Completar Implementación de Docker (CRÍTICO)

Deben tener la aplicación completamente funcional en Docker para la segunda entrega. Esto incluye:
- Base de datos PostgreSQL
- Backend Django
- Frontend React
- Todos los servicios comunicándose correctamente

### 2. Implementar Sistema de Reseñas

```python
class ProductReview(models.Model):
    product = models.ForeignKey(Product, on_delete=models.CASCADE, related_name='reviews')
    user = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.CASCADE)
    rating = models.PositiveSmallIntegerField(
        validators=[MinValueValidator(1), MaxValueValidator(5)]
    )
    comment = models.TextField()
    created_at = models.DateTimeField(auto_now_add=True)
    
    class Meta:
        unique_together = ('product', 'user')
    
    def save(self, *args, **kwargs):
        super().save(*args, **kwargs)
        # Recalcular rating promedio del producto
        avg_rating = self.product.reviews.aggregate(Avg('rating'))['rating__avg']
        self.product.rating = round(avg_rating, 1) if avg_rating else 0
        self.product.save(update_fields=['rating'])
```

### 3. Agregar Sistema de Wishlist

```python
class Wishlist(models.Model):
    user = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.CASCADE)
    products = models.ManyToManyField(Product, related_name='wishlisted_by')
    created_at = models.DateTimeField(auto_now_add=True)
    
    class Meta:
        unique_together = ('user',)
```

### 4. Implementar Notificaciones por Email

```python
from django.core.mail import send_mail
from django.template.loader import render_to_string

def send_order_confirmation_email(order):
    context = {
        'order': order,
        'items': order.items.all(),
        'user': order.user,
    }
    
    html_message = render_to_string('emails/order_confirmation.html', context)
    
    send_mail(
        subject=f'Confirmación de compra - Orden #{order.id}',
        message='',
        html_message=html_message,
        from_email='noreply@codeacademy.com',
        recipient_list=[order.user.email],
    )

def send_certificate_email(certificate):
    send_mail(
        subject=f'¡Felicitaciones! Certificado de {certificate.product.title}',
        message=f'Has completado el curso {certificate.product.title}',
        from_email='noreply@codeacademy.com',
        recipient_list=[certificate.user.email],
    )
```

### 5. Agregar Tests Automatizados

```python
from django.test import TestCase
from decimal import Decimal

class ProductModelTest(TestCase):
    def test_course_cannot_have_pages(self):
        product = Product(
            title="Test Course",
            type=Product.TYPE_COURSE,
            pages=100  # Esto debería fallar
        )
        with self.assertRaises(ValidationError):
            product.full_clean()
    
    def test_book_download_limit(self):
        download = BookDownload.objects.create(
            user=self.user,
            product=self.book,
            download_count=3,
            max_downloads=3
        )
        self.assertFalse(download.download_count < download.max_downloads)
```

### 6. Optimizar Generación de Certificados

Si usan ReportLab para generar PDFs:

```python
from reportlab.lib.pagesizes import letter, landscape
from reportlab.pdfgen import canvas
from io import BytesIO

def generate_course_certificate(user, product):
    buffer = BytesIO()
    pdf = canvas.Canvas(buffer, pagesize=landscape(letter))
    width, height = landscape(letter)
    
    # Diseño del certificado
    pdf.setFont("Helvetica-Bold", 36)
    pdf.drawCentredString(width/2, height-100, "CERTIFICADO DE FINALIZACIÓN")
    
    pdf.setFont("Helvetica", 18)
    pdf.drawCentredString(width/2, height-200, f"Otorgado a: {user.get_full_name()}")
    
    pdf.setFont("Helvetica", 14)
    pdf.drawCentredString(width/2, height-250, f"Por completar el curso:")
    
    pdf.setFont("Helvetica-Bold", 20)
    pdf.drawCentredString(width/2, height-290, product.title)
    
    pdf.save()
    buffer.seek(0)
    return buffer
```

---

## Aspectos Positivos del Proyecto

- Modelado de datos sofisticado y bien pensado
- Separación en apps Django (buena arquitectura)
- Uso de DRF con filtros y búsqueda avanzada
- Optimización de queries con select_related/prefetch_related
- Sistema de progreso de cursos implementado
- Control de descargas de libros
- Generación de certificados
- Integración con Stripe para pagos
- Documentación detallada en README

---

## Próximos Pasos

1. **CRÍTICO: Implementar Docker completo** - Sin esto no cumple la rúbrica
2. **Agregar control de permisos** en endpoints de descarga
3. **Implementar webhook de Stripe** correctamente
4. **Agregar tests** para modelos y vistas críticas
5. **Implementar sistema de reseñas** para productos
6. **Agregar notificaciones por email** para órdenes y certificados
7. **Optimizar performance** con caché para catálogo

Su proyecto tiene una arquitectura sólida y modelos bien diseñados. El principal problema es la falta de Docker funcional, que es requisito fundamental. Para la segunda entrega, enfóquense en completar Docker, agregar las funcionalidades sugeridas y asegurar que todo el flujo de compra funcione end-to-end.

---

*Evaluación realizada sobre el código fuente del repositorio GitHub.*
