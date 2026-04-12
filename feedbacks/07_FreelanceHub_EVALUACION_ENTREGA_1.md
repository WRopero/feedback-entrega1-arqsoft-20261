# Evaluacion Tecnica - Professional Services Marketplace (FreelanceHub)

**Estudiantes:** Andres Velez Rendon, Tomas Ramirez, Felipe Gomez
**Repositorio:** `AndresVelezR/professional-services-marketplace`
**Nota: 3.71 / 5.0**

---

## 1. Descripcion del Producto y README

**Lo que esta bien:**
- README claro con diagrama de arquitectura, instrucciones Docker y local
- Stack bien definido: Django 6 + DRF + PostgreSQL + Next.js 16 + TypeScript + shadcn/ui
- Seed data con credenciales demo y comando para superusuario

**Que mejorar:**
- La pagina raiz (`/`) muestra una galeria de componentes de shadcn en vez del producto real
- Metadata de Next.js sin actualizar ("Create Next App")

---

## 2. Modelos de Django

**Lo que esta bien:**
- `Usuario` extiende `AbstractUser` con email como campo de auth y UUIDs
- Relacion `Perfil` OneToOne, `Habilidad` M2M, `Experiencia` FK con cascade
- `PublicacionManager` con custom queryset methods
- Indices en `Publicacion` para queries frecuentes
- `ImagenPublicacion` con upload dinamico y orden
- Todos los modelos con `__str__`, `verbose_name`, `TextChoices`

**Que mejorar:**
- `imagen_url` en `Publicacion` es redundante con `ImagenPublicacion` - dos fuentes de imagenes
- Metodo `actualizar_perfil` en `Perfil` nunca se usa - codigo muerto
- Contratos, resenas y chat no existen en `main`. La rama `feat/chat-contratos` los tiene implementados (19 commits, ~3,365 lineas) pero nunca fue mergeada

---

## 3. API REST

**Lo que esta bien:**
- Uso de generics de DRF, permiso personalizado `EsCreadorOSoloLectura`
- Paginacion, filtros por busqueda/categoria/precio/orden
- `select_related`/`prefetch_related` para evitar N+1
- Soft delete en publicaciones
- Serializers diferenciados list/detail/create
- `@transaction.atomic` en registro
- Separacion read/write con `habilidad_ids` write-only

**Que mejorar:**
- Filtros de precio no validan tipo numerico (crash con strings)
- `PUT` con `partial=True` es semantica de `PATCH`
- `ExperienciaDetailView` maneja manualmente lo que generics ya resuelve
- Password solo valida `min_length=8`, no usa los validators de Django
- No hay `DEFAULT_PERMISSION_CLASSES` - por defecto todo es `AllowAny`

---

## 4. Frontend

**Lo que esta bien:**
- Arquitectura feature-based con models/services/hooks/components por feature
- Patron adapter para desacoplar respuestas de API de la UI
- Custom hooks para data fetching
- Auth guard con route groups `(protected)`
- shadcn/ui, Biome, TypeScript estricto
- Debounce en busqueda, estados de loading/error/empty

**Que mejorar:**
- Refresh token se descarta - usuarios pierden sesion despues de 1 hora
- Contratos y dashboard usan datos hardcodeados (mock)
- Seccion de contratos esta en ingles, el resto en espanol
- Filtros de "Tiempo de Entrega" y "Calificacion" no funcionan
- Usa `<img>` en vez de `<Image>` de Next.js
- Tipo `Perfil` duplicado entre auth y profile

---

## 5. Docker

**Lo que esta bien:**
- 3 servicios con red dedicada, healthcheck en PostgreSQL
- `depends_on` con `condition: service_healthy`
- Volume persistente, entrypoint con migraciones automaticas

**Que mejorar:**
- `SECRET_KEY` insegura como default y no se configura en docker-compose
- Usa servidor de desarrollo tanto en backend (`runserver`) como frontend (`bun dev`)
- No hay `.dockerignore`

---

## 6. Commits y Equipo

**Lo que esta bien:**
- 87 commits con Conventional Commits, ramas feature y PRs

**Problemas:**
- Distribucion muy desigual: Tomas 72 commits (82.7%), Andres 15 (17.3%), Felipe 0 (0%)
- Rama `feat/chat-contratos` con funcionalidad core nunca mergeada a `main`

---

## 7. Analisis SOLID

### SRP (Single Responsibility Principle)

Un modulo debe tener una sola razon de cambio. Si un archivo cambia cuando modifico autenticacion Y cuando modifico experiencias laborales, tiene dos razones de cambio.

- Las apps estan bien separadas por dominio y los serializers por operacion
- `usuarios/views.py` mezcla registro, perfil, habilidades y experiencias en un solo archivo. Si un equipo trabaja en autenticacion y otro en experiencias, ambos modifican el mismo archivo
- `AuthContext.tsx` mezcla manejo de tokens y datos de perfil. Si cambia la estrategia de auth (pasar de localStorage a cookies), tambien se afecta la carga de perfil

### OCP (Open/Closed Principle)

El software debe estar abierto para extension, cerrado para modificacion. Un ejemplo clasico: si tengo tipos de pago como enum (`TARJETA`, `EFECTIVO`), agregar `BITCOIN` requiere modificar el enum. Si en cambio tengo un modelo `MetodoPago` en la base de datos, agrego registros sin tocar codigo.

- `TextChoices` para estados permite extensibilidad
- Categorias hardcodeadas como enum en el modelo. Agregar una categoria nueva (ej: "Inteligencia Artificial") requiere modificar el codigo fuente, hacer migracion y redesplegar. Deberia poder agregarse sin tocar codigo

### LSP (Liskov Substitution Principle)

- Sin violaciones evidentes. Los modelos no tienen herencia profunda

### ISP (Interface Segregation Principle)

Un cliente no deberia depender de metodos que no usa. Si un endpoint solo necesita nombre y email, no deberia recibir un serializer con 13 campos incluyendo foto, habilidades y experiencias.

- `PerfilSerializer` sirve para todo: lectura, escritura, listado. Un endpoint que solo necesita mostrar el nombre del creador de una publicacion carga todo el perfil completo
- Dos interfaces `Perfil` diferentes en el frontend para el mismo concepto, una en auth y otra en profile

### DIP (Dependency Inversion Principle)

Los modulos de alto nivel no deben depender de modulos de bajo nivel, ambos deben depender de abstracciones. Si una vista llama directamente a `MiModelo.objects.create()`, esta acoplada a Django ORM. Si manana necesito agregar una notificacion por email al crear, tengo que modificar la vista.

- No hay capa de servicios. Las vistas llaman directamente al ORM. Toda la logica de negocio vive en views y serializers
- En la rama de contratos, aceptar una propuesta importa y crea directamente objetos de la app de chat dentro del metodo de la vista, acoplando los dominios de contratos y chat

---

## 8. Resumen de Issues Criticos

| # | Issue | Impacto |
|---|-------|---------|
| 1 | Rama `feat/chat-contratos` no mergeada | Funcionalidad core no llega al producto |
| 2 | Contratos/Dashboard con datos mock | 3 de 5 secciones del sidebar no funcionan |
| 3 | No hay capa de servicios (DIP) | Vistas acopladas al ORM |
| 4 | No se usa refresh token | Sesion se pierde despues de 1 hora |
| 5 | Un miembro con 0 commits | Sin evidencia de contribucion |

---

## 9. Requisitos para Entrega 2 (Rubrica)

### 1. Correcciones de la Parte 1 (10%)

- [ ] Mergear contratos/chat a `main` y reemplazar datos mock con API real
- [ ] Consistencia de idioma en toda la app
- [ ] Implementar refresh token
- [ ] Eliminar `SECRET_KEY` hardcodeada
- [ ] Agregar `DEFAULT_PERMISSION_CLASSES`
- [ ] Corregir pagina raiz y metadata

### 2. Diagrama de arquitectura actualizado (5%)

Entregar diagrama que refleje la arquitectura actual del sistema: capas (presentacion, servicios, dominio, persistencia), componentes principales, y las interfaces abstractas (DIP). Formato legible (draw.io, Mermaid, PlantUML).

### 3. Servicios implementados (30%)

Servicio de registro de usuarios, servicio de gestion de contratos/propuestas, servicio de notificaciones (email al aceptar propuesta)

Cada servicio debe:
- Estar en un archivo `services.py` dentro de su app
- Encapsular la logica de negocio (las vistas solo delegan)
- Usar `@transaction.atomic` donde haya operaciones de escritura multiples

### 4. Inversion de dependencias (15%)

Crear una interfaz abstracta para el sistema de notificaciones, con al menos dos implementaciones concretas (una real y una mock). Los servicios deben depender de la abstraccion, no de implementaciones concretas.

### 5. Pruebas unitarias (10%)

Implementar al menos **dos pruebas unitarias** que verifiquen logica de negocio central del marketplace.

### 6. Calidad del codigo y arquitectura (15%)

- Separacion clara en capas (vistas -> servicios -> modelos)
- Sin logica de negocio en vistas
- Sin imports circulares entre apps
- Sin secretos en el codigo fuente
- Todos los miembros del equipo deben tener commits en el repositorio

### 7. Despliegue en nube + sistema de dos idiomas (15%)

- **Despliegue**: El proyecto debe estar desplegado y accesible en un servicio cloud (Railway, Render, AWS, GCP, Azure, etc.)
- **Internacionalizacion (i18n)**: Implementar soporte para **dos idiomas** (espanol + ingles)

---

*Evaluacion realizada sobre el codigo fuente del repositorio `AndresVelezR/professional-services-marketplace`, rama `main` y rama `feat/chat-contratos`.*
