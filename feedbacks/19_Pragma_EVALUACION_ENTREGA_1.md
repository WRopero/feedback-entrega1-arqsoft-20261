# Evaluación Técnica - Pragma

---

## Análisis del Código Implementado

El proyecto Pragma es un sistema de procesamiento de facturas con OCR implementado con Django, PostgreSQL y Docker, con capacidades reales de extracción de texto mediante PyMuPDF y Tesseract. El equipo (David Cossio Salazar, Sara Echeverri Gomez, Santiago Arboleda Giraldo) entregó una implementación técnicamente sólida y significativamente más elaborada de lo que suele verse en una primera entrega: capa de servicios bien estructurada, pruebas unitarias comprehensivas, OCR funcional con fallback entre motores, y lógica de comparación de pagos con scoring ponderado. **El punto crítico del proyecto no es la calidad del código sino el historial de Git: todo el trabajo fue committeado en un único commit, lo que impide verificar la distribución de trabajo entre los integrantes del equipo.** Esta situación ya impactó la calificación de la entrega anterior y debe corregirse de forma estructural en la Entrega 2.

---

### Modelos de Datos

**Archivos revisados:**
- `pragma/core/models/cliente.py`
- `pragma/core/models/factura.py`
- `pragma/core/models/certificado_bancario.py`
- `pragma/core/models/detalle_pago.py`
- `pragma/core/models/usuario.py`

El diseño de modelos es modular y bien documentado. La organización en sub-paquete `models/` con un archivo por entidad es una práctica de ingeniería que favorece la legibilidad y el mantenimiento, especialmente cuando los modelos crecen en complejidad.

**Fortalezas identificadas:**

1. **Índices de base de datos explícitos donde importan**: `Factura` define `db_index=True` en `numero_factura`, `fecha` y `cliente_nit`, que son exactamente los campos por los que el sistema realiza búsquedas de matching. Esta decisión demuestra conciencia del rendimiento a nivel de base de datos, algo inusual en proyectos académicos:

```python
class Factura(models.Model):
    numero_factura = models.CharField(max_length=100, unique=True, db_index=True)
    fecha          = models.DateField(db_index=True)
    cliente_nit    = models.CharField(max_length=20, db_index=True)
    monto          = models.DecimalField(max_digits=15, decimal_places=2)
    ocr_data       = models.JSONField(null=True, blank=True)
```

2. **`ocr_data` como `JSONField`**: Almacenar los datos extraídos por OCR directamente en el modelo como JSON es una decisión pragmática correcta: permite guardar la estructura completa de lo que el OCR interpretó sin requerir un esquema adicional, y facilita la auditoría y el re-procesamiento sin migraciones adicionales.

3. **`DetallePago` como tabla de matching explícita**: La entidad `DetallePago` con `match_score` (DecimalField), `estado_match` y `diferencias` convierte el resultado del algoritmo de comparación en un registro persistente, lo que es la arquitectura correcta para este tipo de sistema. El resultado del matching no es efímero — tiene valor histórico y de auditoría.

4. **`unique=True` en `Cliente.nit` y `Meta.ordering` definido**: El campo `nit` con `unique=True` garantiza integridad referencial a nivel de base de datos, no solo en Python. La definición de `Meta.ordering` en los modelos evita la necesidad de especificar ordenamiento en cada consulta del ORM.

**Áreas de mejora:**

1. **`SECRET_KEY` hardcodeado en `docker-compose.yml`**: El `SECRET_KEY` de Django y posiblemente otras credenciales están definidos directamente en el archivo `docker-compose.yml`, que está bajo control de versiones. Esto es un problema de seguridad que debe corregirse antes de cualquier despliegue real:

```yaml
# Actual — inseguro, secret key en texto plano en el repositorio
environment:
  - SECRET_KEY=django-insecure-...
  - DATABASE_URL=postgresql://...

# Correcto — usar archivo .env (no committeado) con .env.example como referencia
env_file:
  - .env
```

El repositorio debe incluir un archivo `.env.example` con los nombres de las variables pero sin valores reales, y `.env` debe estar en `.gitignore`.

---

### Servicios y Lógica de Negocio

**Archivos revisados:**
- `pragma/core/services/comparador_pagos.py`
- `pragma/core/services/ocr_service.py`
- `pragma/core/services/dashboard_service.py`
- `pragma/core/services/export_service.py`

Esta es la parte más destacable del proyecto. La existencia de una capa de servicios real, con funciones bien definidas, separa correctamente la lógica de negocio de las vistas y del ORM. Es precisamente lo que se espera en esta asignatura y el equipo lo implementó de forma proactiva.

**Fortalezas identificadas:**

1. **Scoring ponderado con lógica de negocio clara en `comparador_pagos.py`**: El algoritmo de comparación asigna penalizaciones específicas y justificadas por tipo de discrepancia:

```python
def comparar_pagos(factura, certificado):
    score = 100
    diferencias = []

    if monto_diferente:
        score -= 40           # El monto es el criterio más crítico
        diferencias.append(...)
    if nit_diferente:
        score -= 30           # NIT es identificador del cliente
        diferencias.append(...)
    if diferencia_fecha > 2:  # días
        score -= 30           # Tolerancia de 2 días para procesamiento bancario
        diferencias.append(...)

    return {
        'estado_match': determinar_estado(score),
        'diferencias': diferencias,
        'match_score': Decimal(score),
        'resumen': generar_resumen(score, diferencias)
    }
```

La separación de pesos refleja conocimiento del dominio: una diferencia de monto es más grave que una diferencia de fecha en el procesamiento de facturas, y asignar un número al razonamiento hace el criterio auditable.

2. **OCR con fallback real entre PyMuPDF y Tesseract**: `ocr_service.py` implementa un pipeline de extracción con dos niveles: primero intenta extracción de texto nativo con PyMuPDF (para PDFs con texto embebido) y, si el resultado está vacío o es insuficiente, aplica Tesseract con preprocesamiento en escala de grises para documentos escaneados. Este patrón de fallback es correcto y realista dado que en la práctica los documentos llegan en formatos mixtos.

3. **`crear_o_actualizar_detalle_pago()` es idempotente con `update_or_create`**: La función usa `update_or_create` de Django, lo que garantiza que ejecutar el matching múltiples veces sobre el mismo par factura-certificado produce el mismo resultado sin duplicar registros. Este es exactamente el tipo de garantía que un sistema de procesamiento financiero necesita, y el hecho de que exista un test que verifica esta propiedad es doblemente correcto.

4. **Búsqueda de candidatos con matching fuzzy por monto y fecha**: Las funciones `buscar_factura_candidata` y `buscar_certificado_candidato` usan proximidad de monto y rango de fechas en lugar de igualdad exacta, lo que es realista dado que los documentos procesados con OCR tienen variabilidad inherente en la extracción numérica.

**Áreas de mejora:**

1. **`import uuid` dentro del cuerpo de una función en `admin_views.py`**: Los imports deben estar al inicio del módulo, no dentro de funciones. Los imports dentro de funciones solo tienen justificación en casos muy específicos (imports circulares, imports condicionados con alto costo de carga). Este debe moverse al encabezado del archivo:

```python
# Actual — import dentro de función (no estándar)
def alguna_vista(request):
    import uuid
    transaction_id = str(uuid.uuid4())

# Correcto — import al inicio del módulo
import uuid

def alguna_vista(request):
    transaction_id = str(uuid.uuid4())
```

2. **Import adicional a nivel de función en `admin_views.py` (~línea 44)**: Similar al punto anterior, cualquier import que deba usarse en múltiples partes de un módulo debe estar al nivel superior del archivo. Revisar el módulo completo en busca de otros imports mal ubicados.

---

### Vistas y Seguridad

**Archivos revisados:**
- `pragma/core/views/admin_views.py`
- `pragma/core/views/usuario_views.py`
- `pragma/core/views/permissions.py`

**Fortalezas identificadas:**

1. **Flujo de carga con revisión antes de confirmar**: El flujo de subida de documentos sigue el patrón upload → OCR → revisión en sesión → confirmación → auto-trigger de matching, lo que permite al usuario validar la extracción antes de persistir el registro. Es un flujo de UX correcto para un sistema que depende de OCR, donde los errores de extracción son esperables y no deben comprometer la integridad de los datos.

2. **`permissions.py` con `ensure_admin_or_raise(request)`**: Centralizar la verificación de permisos en una función reutilizable es mejor práctica que repetir la verificación en cada vista. Esto facilita cambiar la lógica de autorización en un único lugar y reduce el riesgo de que alguna vista quede desprotegida por omisión.

3. **Separación entre `admin_views.py` y `usuario_views.py`**: Tener vistas de administración y vistas de usuario en módulos separados refleja correctamente los dos contextos de uso del sistema y facilita aplicar diferentes políticas de autorización a cada conjunto de endpoints.

---

### Pruebas Unitarias

**Archivo revisado:** `pragma/core/tests/test_services.py`

El proyecto tiene una suite de pruebas comprehensiva que cubre tanto casos positivos como negativos y tanto la capa de servicios como la integración con el ORM. La presencia de tests en una primera entrega, especialmente con esta profundidad, es notable.

**Cobertura actual:**
- `test_parse_invoice_text_extracts_fields` — extracción básica de campos desde texto OCR
- `test_parse_invoice_text_with_complex_labels` — etiquetas en español complejas
- `test_parse_invoice_text_with_spanish_long_date` — parsing de "24 de Marzo de 2026"
- `test_extract_invoice_data_with_unsupported_format` — manejo de formato no soportado
- `test_comparar_pagos_full_match` / `test_comparar_pagos_no_match` — extremos del scoring
- `test_dashboard_metrics` — métricas de agregación
- `test_export_services_generate_bytes` — generación de PDF y Excel
- `test_crear_o_actualizar_detalle_pago_is_idempotent` — propiedad de idempotencia
- `test_buscar_certificado_candidato` — lógica de matching fuzzy

Esta cobertura indica que el equipo entendió que las pruebas no son solo verificación de "que funciona" sino documentación ejecutable del comportamiento esperado del sistema, incluyendo sus invariantes (idempotencia).

---

### Infraestructura y Despliegue

**Archivos revisados:**
- `Dockerfile`
- `docker-compose.yml`

**Fortalezas identificadas:**

1. **Dockerfile con usuario no-root**: La creación de un usuario sin privilegios de root en el contenedor es una práctica de seguridad correcta que la mayoría de proyectos omiten. Reduce la superficie de ataque si el proceso web es comprometido y es una buena práctica para entornos de producción.

2. **Instalación de dependencias del sistema para OCR**: El `Dockerfile` instala `tesseract-ocr` y `libgl1` como dependencias del sistema, lo que garantiza que el entorno es completamente reproducible y no depende de que el sistema anfitrión tenga estas librerías instaladas. La imagen es autocontenida.

3. **Healthcheck en el servicio de base de datos**: El `docker-compose.yml` define un healthcheck para postgres:15, lo que garantiza que el contenedor web espera a que la base de datos esté lista para aceptar conexiones antes de intentar la primera migración.

---

### Patrones y Principios SOLID

**Lo que funciona bien:**
- La capa de servicios (`services/`) está claramente separada de las vistas y del ORM, aplicando el Principio de Responsabilidad Única de forma consistente a lo largo de todo el proyecto.
- `comparar_pagos` es esencialmente una función pura: dado el mismo input, siempre produce el mismo output. Esto la hace trivialmente testeable y razonable de auditar.
- La modularización de modelos en archivos individuales aplica el Principio de Segregación de Interfaces al nivel de módulo — cada archivo tiene exactamente una razón para cambiar.
- `crear_o_actualizar_detalle_pago` encapsula completamente la lógica de persistencia del matching, evitando que las vistas necesiten conocer los detalles de `update_or_create`.
- El modelo de dominio usa `ForeignKey(Cliente, SET_NULL)` en `Factura`, lo que preserva el historial de facturas aunque se elimine el cliente — decisión correcta para un sistema contable.

**Problemas arquitectónicos:**

1. **Historial de Git sin distribución de trabajo verificable**: Un único commit con toda la implementación es el problema más grave del proyecto desde la perspectiva de evaluación. No permite determinar quién implementó qué componente, lo que hace imposible evaluar la contribución individual de cada integrante. Para la Entrega 2, cada miembro del equipo debe tener commits atribuidos a su nombre con sus credenciales de Git configuradas correctamente desde este punto en adelante.

2. **Credenciales en el repositorio**: El `SECRET_KEY` en `docker-compose.yml` es una vulnerabilidad de seguridad. En un proyecto real, esto implicaría rotar la clave de inmediato. Aunque en contexto académico el impacto operacional es menor, establecer el hábito correcto ahora es parte del aprendizaje de esta asignatura.

---

## Requisitos para Entrega 2 (Rúbrica)

### 1. Correcciones de la Parte 1 (10%)

- [ ] Mover el `SECRET_KEY` y `DATABASE_URL` fuera de `docker-compose.yml` y hacia un archivo `.env` no committeado
- [ ] Agregar `.env` al `.gitignore` y crear `.env.example` con los nombres de las variables requeridas pero sin valores reales
- [ ] Mover todos los `import` que están dentro de cuerpos de funciones al inicio del módulo correspondiente en `admin_views.py`
- [ ] Verificar que cada integrante del equipo tiene commits atribuidos con sus credenciales de Git correctas desde este punto en adelante — esto es un requisito de evaluación, no solo de buenas prácticas

### 2. Diagrama de arquitectura actualizado (5%)

- [ ] Diagrama que muestre el flujo completo: subida de archivo → OCR service → revisión en sesión → persistencia → auto-matching
- [ ] Incluir la relación entre `Factura`, `CertificadoBancario` y `DetallePago` con cardinalidades
- [ ] Mostrar los dos actores del sistema: administrador y usuario, con sus flujos diferenciados y los permisos que los separan
- [ ] Indicar explícitamente en qué capa vive cada responsabilidad (vista, servicio, modelo)

### 3. Servicios implementados (30%)

El proyecto ya cuenta con una base de servicios sólida. Para la Entrega 2 se requiere formalizar y expandir:

- [ ] Documentar con docstrings completos todos los métodos públicos de `comparador_pagos.py`, `ocr_service.py`, `dashboard_service.py` y `export_service.py` que aún no los tengan, especificando parámetros, tipo de retorno y excepciones posibles
- [ ] Agregar manejo explícito de errores en `ocr_service.py` para casos donde PyMuPDF y Tesseract fallen simultáneamente, retornando un resultado estructurado con el campo `errors` poblado en lugar de lanzar una excepción no manejada
- [ ] Expandir `dashboard_service.py` con métricas adicionales: tasa de match exitoso sobre el total, distribución de rangos de score, y facturas sin ningún certificado candidato en el sistema
- [ ] Confirmar que ninguna vista ejecuta lógica de negocio directamente — toda operación de dominio debe pasar por un servicio

### 4. Inversión de dependencias (15%)

- [ ] Los servicios de alto nivel no deben instanciar sus dependencias internamente — deben recibirlas como parámetros para permitir testing con mocks sin necesidad de base de datos
- [ ] `comparar_pagos(factura, certificado)` ya cumple este principio correctamente — extender el mismo patrón a cualquier función en los servicios que actualmente acceda al ORM de forma directa sin recibirlo como parámetro
- [ ] Si se agregan nuevos servicios para la Entrega 2, definir primero sus firmas (usando `Protocol` de Python o clases base abstractas) antes de implementar, para explicitar el contrato

### 5. Pruebas unitarias (10%)

La suite de tests existente es sólida. Para la Entrega 2:

- [ ] Agregar tests para el flujo de carga completo (upload → extracción OCR → revisión en sesión → confirmación de guardado) usando el `Client` de Django
- [ ] Agregar tests negativos para `buscar_factura_candidata` cuando no existe ningún candidato viable en la base de datos
- [ ] Agregar tests de integración que verifiquen que las vistas llaman a los servicios correctos y no contienen lógica de negocio inline
- [ ] Mantener y ampliar la cobertura existente — no reducirla al incorporar nueva funcionalidad

### 6. Calidad del código y arquitectura (15%)

- [ ] Todos los imports al nivel de módulo — cero imports dentro de cuerpos de funciones
- [ ] Variables de entorno manejadas exclusivamente vía `os.environ` o `django-environ`, nunca como literales en archivos committeados
- [ ] Revisión de consistencia en nombres de variables y funciones — mezcla de inglés y español debe resolverse a favor del español para términos de dominio
- [ ] Sin código comentado ni print statements de debugging en el código final entregado

### 7. Despliegue en nube + sistema de dos idiomas (15%)

- [ ] URL pública funcional del proyecto desplegado con el sistema OCR completamente operativo (subida de archivo y extracción funcionando en el entorno de nube)
- [ ] La interfaz debe estar disponible en español e inglés usando `django.utils.translation`
- [ ] Archivos `.po` / `.mo` generados con `makemessages` y `compilemessages` para ambos idiomas
- [ ] `LANGUAGE_CODE`, `USE_I18N = True` y `LocaleMiddleware` configurados correctamente en `settings.py`
- [ ] El selector de idioma debe ser visible en la interfaz de usuario tanto en el panel de administración como en la vista del usuario final
