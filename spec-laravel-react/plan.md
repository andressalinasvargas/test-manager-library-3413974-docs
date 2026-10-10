# Plan de Implementación — Manager Library (PHP Laravel + React)

Plan maestro de ejecución del proyecto **Manager Library** mediante la metodología **Spec-Driven Development (SDD)**, articulando las fases de construcción en **PHP Laravel** y **React con Vite**.

---

## 1. Estrategia y Enfoque de Desarrollo

El desarrollo se fundamenta en los siguientes pilares:
1. **El Spec como Única Fuente de Verdad:** Las decisiones arquitectónicas, modelos relacionales y contratos HTTP no se improvisan; se traducen estrictamente desde las especificaciones formales.
2. **Desacoplamiento Estricto:** Construcción de adentro hacia afuera (Dominio → Puertos → Servicios de Aplicación → Adaptadores e Infraestructura).
3. **Evidencia Ejecutable (Artículo XII):** Cada fase de tareas produce un resultado comprobable y reproducible mediante contenedores Docker, pruebas unitarias en PHP y scripts de verificación.

---

## 2. Cronograma de Fases y Tareas (T-01 a T-23)

### Fase 1: Cimientos de Infraestructura y Base de Datos
- **T-01 — Entorno de Ejecución y Health Check:**
  - Configuración del esqueleto de la aplicación en PHP con Laravel 11.
  - Implementación del endpoint `GET /api/v1/health` que devuelve `{"status":"ok"}`.
  - Creación del archivo `docker-compose.yml` para levantar PostgreSQL 16 y la API en un solo comando (**T-20**, **RNF-5**).
- **T-02 — Migración Inicial en Laravel:**
  - Creación de la migración en `database/migrations/` definiendo el esquema `library`.
  - Definición de las seis tablas en singular: `category`, `book`, `member`, `user`, `loan`, `loan_item`.
  - Configuración de claves foráneas (**FK-1** a **FK-6**), restricciones de chequeo `CHECK` (**RN-1**, **ADR-002**) e índices únicos parciales (**ADR-003**).
  - Población de las 5 categorías inmutables con sus UUIDs fijos (**D-01** a **D-05**).

### Fase 2: Núcleo de Dominio Puro en PHP
- **T-03 — Objetos de Valor (`Isbn`, `Quantity`, `DateRange`):**
  - Implementación de `Isbn` con validación matemática de 13 dígitos numéricos (**RN-4**).
  - Implementación de `Quantity` asegurando enteros positivos para stock.
  - Pruebas unitarias sin dependencias de Laravel ni de base de datos (**Artículo I**).
- **T-04 — Agregado `Book`:**
  - Métodos `withdraw()` y `restock()` asegurando que `available_copies` se mantenga entre 0 y `total_copies`.
  - Implementación de baja lógica mediante `markDeleted()` preservando histórico.
- **T-05 — Agregado `Loan` y `LoanItem`:**
  - Validación del límite de 1 a 3 libros por operación (**RN-3**).
  - Asignación estricta de 14 días para la fecha de devolución (`due_at`) (**RN-2**).
  - Cálculo dinámico de estados (`open`, `overdue`, `closed`) mediante la interfaz `Clock` (**Artículo VII**).
- **T-06 — Entidades `Member` y `User`:**
  - `Member`: validación de documento de identidad único y control de elegibilidad de préstamo (**HU-05**).
  - `User`: normalización de nombre de usuario en minúsculas y roles permitidos (`admin`, `librarian`) (**HU-01**, **HU-02**).

### Fase 3: Puertos y Casos de Uso en Aplicación
- **T-07 — Autenticación y Usuarios:**
  - `LoginService`: verificación de credenciales con `PasswordHasher` y generación de tokens de 8 horas con `TokenIssuer` (**HU-01**, **CA-01.1**, **CA-01.2**).
  - `CreateLibrarianService`: creación de bibliotecarios por el administrador (**HU-02**, **CA-02.1**, **CA-02.2**).
- **T-08 — Gestión de Libros:**
  - `BookService`: creación, actualización, baja lógica y búsqueda de libros (**HU-03**, **HU-04**, **CA-03.1**, **CA-04.1**).
- **T-09 — Préstamos de Libros:**
  - `LoanService`: validación de stock disponible, bloqueo por moras activas (**CA-06.1**), límite de préstamos simultáneos (**CA-06.2**) y descuento atómico de inventario.
- **T-10 — Devoluciones y Préstamos Vencidos:**
  - Retorno unitario de líneas de préstamo, restitución de copias y listado de préstamos en mora (**HU-07**, **HU-09**, **CA-07.1**, **CA-09.1**).
- **T-11 — Reporte de Más Prestados:**
  - `ReportService`: generación del ranking Top 10 con datos congelados del momento del préstamo (**HU-10**, **CA-10.1**, **ADR-004**).

### Fase 4: Adaptadores de Infraestructura y Transporte en Laravel
- **T-12 y T-13 — Repositorios PostgreSQL:**
  - Adaptadores con consultas SQL optimizadas sobre el esquema `library` evitando Eloquent para máximo control transaccional (**RNF-2**).
- **T-14 — Adaptador de Seguridad JWT y Hash:**
  - Emisión de tokens Bearer firmados con HMAC-SHA256 (`JWT_SECRET`) y hashing seguro de contraseñas.
- **T-15 — Adaptador de Portadas en Disco:**
  - Almacenamiento local de archivos PNG/JPG hasta 2 MB en `storage/app/covers/`.
- **T-16 — Controlador RESTful y Middleware:**
  - Enrutamiento de los 21 endpoints bajo `/api/v1` en `routes/api.php`.
  - Middleware `AuthenticateWithJwt` para validación de tokens y roles.
  - Transformación unificada de errores a formato JSON canónico con los 14 códigos de error.
- **T-17 — Composición y Bootstrap Inicial:**
  - Enlace de puertos e implementaciones en `AppServiceProvider` (**Artículo III**).
  - Comando de Artisan `library:bootstrap-admin` para crear el usuario administrador inicial a partir de las variables `ADMIN_USERNAME` y `ADMIN_PASSWORD` del entorno.

### Fase 5: Frontend React, Orquestación y Herramientas
- **T-20 — Orquestación Docker Compose:**
  - Servicios `db`, `api` y `app` conectados en la red privada `library-net`.
  - Aislamiento completo de variables de entorno mediante `.env` (Artículo IX).
- **T-21 — Sembrador de Demostración (`tool`):**
  - Script en PHP CLI (`seed.php`) que consume exclusivamente la API REST para poblar 1 bibliotecario, 40 libros, 10 socios, 15 préstamos y 3 vencidos.
- **T-22 — Aplicación Frontend en React (`app`):**
  - Interfaz SPA completa desarrollada con React y Vite: Login, Catálogo, Socios, Préstamos, Devoluciones, Vencidos y Reporte.
- **T-23 — Landing Page Pública (`page`):**
  - Sitio web estático institucional responsive que presenta el producto sin llamadas a la API y sin exponer credenciales.
