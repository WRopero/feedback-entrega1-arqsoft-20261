# Evaluación Técnica - veripay

> **Nota:** El link enviado apuntaba a la wiki del repositorio (`/wiki/Reglas-de-Programacion`) en lugar del repo principal. La evaluación automatizada no encontró el código. El repositorio correcto es `gatehortus/proyecto-arquitectura`.

---

## Análisis del Código Implementado

El proyecto implementa una plataforma de conciliación de pagos financieros. Es un dominio técnico exigente y el equipo lo abordó con una separación de apps coherente.

### Arquitectura General

Cinco apps con responsabilidades diferenciadas: `certificados`, `conciliacion`, `pagos`, `facturas` y `core`. El modelo de dominio es apropiado para el problema de reconciliación financiera.

### Modelos de Datos

`ProcesoReconciliacion` está bien estructurado con `TextChoices` para estados:

```python
class EstadoProceso(models.TextChoices):
    PENDIENTE   = 'PENDIENTE',   'Pendiente'
    EN_PROCESO  = 'EN_PROCESO',  'En proceso'
    FINALIZADO  = 'FINALIZADO',  'Finalizado'
    ERROR       = 'ERROR',       'Error'
```

Uso de UUID como PK en todos los modelos y campos de auditoría `created_at` / `finalizado_at`. Los contadores `total_facturas`, `facturas_conciliadas`, `facturas_pendientes` permiten seguimiento del progreso sin queries adicionales.

### Capa de Servicios

La app `conciliacion` tiene `services.py`, lo que indica que la lógica de negocio no vive directamente en las vistas.

---

## Áreas de Mejora

**1. Acoplamiento directo entre apps vía imports de modelos**

`conciliacion/models.py` importa directamente de tres apps distintas:

```python
from facturas.models import Factura
from pagos.models import RegistroPago
from certificados.models import CertificadoBancario
```

Esto crea un grafo de dependencias rígido. Si se mueve `Factura` a otro módulo, `conciliacion` se rompe. En contextos financieros, las apps deben comunicarse a través de interfaces o eventos, no imports directos de modelos entre sí.

**Solución:** Definir los Foreign Keys usando strings (`'facturas.Factura'`) en lugar de imports directos, o centralizar las referencias en un módulo `core/models.py`.

**2. Contadores de facturas desincronizables**

Los campos `total_facturas`, `facturas_conciliadas`, `facturas_pendientes` se actualizan manualmente. Si un proceso falla a mitad, los contadores quedan inconsistentes con la realidad.

```python
# Riesgoso
proceso.facturas_conciliadas += 1
proceso.save()

# Correcto — operación atómica
ProcesoReconciliacion.objects.filter(pk=proceso.pk).update(
    facturas_conciliadas=F('facturas_conciliadas') + 1
)
```

**3. Sin tests**

El dominio financiero es exactamente donde los tests son más críticos. Un error en la lógica de conciliación puede implicar que facturas queden sin cruzar o que pagos se apliquen incorrectamente. Para la segunda entrega es obligatorio tener tests de:

- Que una factura conciliada no se vuelve a procesar
- Que la tolerancia de diferencia funciona correctamente
- Que un proceso en estado ERROR no se puede continuar sin reiniciarlo

**4. `archivo_pagos_prueba.csv` en el repositorio**

Datos de prueba financieros no deben estar en el repositorio. Usar un script de generación de datos sintéticos o Django fixtures.

**5. Sin control de acceso visible en vistas**

Las vistas deben verificar que solo usuarios autorizados acceden a los procesos de conciliación. No se observa uso de permisos o decoradores de autenticación consistentes.

---

## Recomendaciones para la Segunda Entrega

1. **Eliminar el acoplamiento entre apps** — usar referencias por string en ForeignKey o un módulo central de tipos
2. **Agregar tests** para los flujos de conciliación — al menos 5 casos de prueba
3. **Proteger operaciones de escritura con transacciones atómicas** — `@transaction.atomic` en el servicio de conciliación
4. **Implementar autenticación** en todos los endpoints
5. **Quitar archivos de datos reales del repositorio** — reemplazar por fixtures o scripts de seed
