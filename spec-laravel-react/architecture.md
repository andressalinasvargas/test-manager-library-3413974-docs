# Arquitectura del Sistema — Manager Library (PHP Laravel + React)

Documento de arquitectura integral de **Manager Library** adaptado a la pila tecnológica oficial del reto técnico: **PHP 8.2+ con Laravel** en el Backend (`api`) y **React con Vite** en el Frontend (`app`), orquestado sobre **PostgreSQL 16** y contenedores **Docker Compose** (`infra`).

---

## 1. Visión General de la Arquitectura

El sistema aplica los preceptos de **Spec-Driven Development (SDD)**, transformando la especificación formal agnóstica en una solución de software escalable, modular y mantenible.

La arquitectura se divide en capas bajo el patrón de **Arquitectura Hexagonal (Puertos y Adaptadores)** en el backend Laravel, complementada con un frontend desacoplado basado en una Single Page Application (SPA) en React:

```
+----------------------------------------------------------------------------------+
|                             CAPA DE PRESENTACIÓN                                 |
|                                                                                  |
|   [ Cliente Web React (Vite SPA) ]             [ Landing Pública (page) ]        |
|        Puerto 5173 / Nginx                           index.html estático         |
+----------------------------------------------------------------------------------+
                                        │
                                        ▼ Peticiones HTTP / JSON (/api/v1)
+----------------------------------------------------------------------------------+
|                            BACKEND PHP (LARAVEL API)                             |
|                                                                                  |
|   ADAPTADORES PRIMARIOS (ENTRADA):                                               |
|   • Rutas API (/api/v1) en routes/api.php                                        |
|   • Middleware de Autenticación Bearer JWT (AuthenticateWithJwt)                 |
|   • Controlador Central RESTful: LibraryApiController                            |
|   • Comandos CLI de Artisan (BootstrapAdminCommand)                              |
|                                                                                  |
|   PUERTOS DE ENTRADA (CASOS DE USO / SERVICIOS):                                 |
|   • LoginService (HU-01)          • CreateLibrarianService (HU-02)               |
|   • BookService (HU-03, HU-04)    • MemberService (HU-05)                        |
|   • LoanService (HU-06, HU-07)    • ReportService (HU-10)                        |
|                                                                                  |
|   NÚCLEO DE DOMINIO PURO (PHP ESTÁNDAR):                                         |
|   • Entidades: Book, Loan, LoanItem, Member, User                                |
|   • Objetos de Valor: Isbn, Quantity, DateRange                                  |
|   • Invariantes y Excepciones: DomainError                                       |
|                                                                                  |
|   PUERTOS SECUNDARIOS (SALIDA):                                                  |
|   • UserRepository            • BookRepository          • MemberRepository       |
|   • LoanRepository            • LoanReader              • ReportReader           |
|   • CategoryRepository        • CoverStorage            • Clock                  |
|   • PasswordHasher            • TokenIssuer             • TransactionRunner      |
|   • IdGenerator                                                                  |
|                                                                                  |
|   ADAPTADORES SECUNDARIOS (INFRAESTRUCTURA):                                     |
|   • PostgresUserRepository    • PostgresBookRepository  • PostgresLoanRepository |
|   • JwtTokenIssuer (HS256)    • PasswordHasherAdapter   • DiskCoverStorage       |
|   • SystemClock               • UuidGenerator           • DbTransactionRunner    |
+----------------------------------------------------------------------------------+
                                        │
                                        ▼ Consultas SQL nativas (PDO)
+----------------------------------------------------------------------------------+
|                         BASE DE DATOS RELACIONAL (INFRA)                         |
|                                                                                  |
|   PostgreSQL 16 Alpine en esquema 'library' (ADR-001, Artículo V)                |
|   Tablas: category, book, member, user, loan, loan_item                          |
+----------------------------------------------------------------------------------+
```

---

## 2. Capas del Backend en Laravel (`api`)

### 2.1 Capa de Dominio (`app/Domain/`)
De acuerdo con el **Artículo I** de la Constitución, el dominio representa el corazón del negocio y está libre de cualquier dependencia de Laravel o paquetes externos.

- **Entidad `Book` (T-04):** Administra el inventario de copias físicas mediante métodos explícitos `withdraw()` (prestar) y `restock()` (devolver). Garantiza la regla de negocio **RN-1** asegurando que `available_copies` nunca sea menor a 0 ni mayor a `total_copies`. Controla la baja lógica marcando `deleted_at` sin eliminar filas físicas.
- **Entidad `Loan` y `LoanItem` (T-05):** Agregado que coordina el préstamo de 1 a 3 libros por operación (regla **RN-3**). La fecha límite de devolución se fija en exactamente 14 días calendario (**RN-2**). El estado del préstamo (`open`, `overdue`, `closed`) y los días de mora se calculan dinámicamente según el **Artículo VII**.
- **Entidad `Member` (T-06):** Representa a los socios lectores identificados por su número de documento único (**D-03**). Controla la validación de préstamos activos y bloquea nuevos préstamos si existen libros vencidos.
- **Entidad `User` (T-06):** Representa al personal del sistema con roles discretos `admin` o `librarian`. Normaliza el nombre de usuario eliminando espacios y forzando minúsculas.
- **Objetos de Valor (T-03):** `Isbn` valida la estructura canónica de 13 dígitos numéricos; `Quantity` asegura valores enteros no negativos; `DateRange` verifica coherencia cronológica.

### 2.2 Capa de Aplicación (`app/Application/`)
Conforme al **Artículo II**, contiene la lógica de orquestación de los casos de uso:

- **Puertos (`app/Application/Ports/`):** Define trece interfaces limpias en PHP:
  1. `Clock`: Abstracción del tiempo para permitir simulaciones y pruebas con `X-Simulated-Time`.
  2. `IdGenerator`: Generación de identificadores únicos universales (UUID v4).
  3. `TransactionRunner`: Ejecución atómica de operaciones transaccionales en base de datos.
  4. `PasswordHasher`: Hashing seguro de contraseñas mediante algoritmos como bcrypt o Argon2id.
  5. `TokenIssuer`: Emisión y validación de tokens Bearer JWT con vigencia de 8 horas (**RN-5**).
  6. `CoverStorage`: Almacenamiento y recuperación de imágenes de portada en disco local (**T-15**).
  7. `UserRepository`: Persistencia y búsqueda de usuarios administradores y bibliotecarios.
  8. `CategoryRepository`: Consulta del catálogo fijo de 5 categorías sembradas (**T-02**).
  9. `BookRepository`: Persistencia, filtrado y modificación concurrente de libros (**T-12**).
  10. `MemberRepository`: Registro y búsqueda de socios por documento de identidad.
  11. `LoanRepository`: Registro transaccional de nuevos préstamos y marcado de devoluciones (**T-13**).
  12. `LoanReader`: Consultas especializadas de préstamos activos, historial y préstamos vencidos (**T-10**).
  13. `ReportReader`: Consulta analítica del Top 10 de libros más prestados (**T-11**, **ADR-004**).

- **Servicios de Aplicación (`app/Application/Services/`):**
  - `LoginService`: Valida credenciales contra `UserRepository` y emite tokens JWT (**HU-01**).
  - `CreateLibrarianService`: Permite al administrador crear bibliotecarios con validación de unicidad (**HU-02**).
  - `BookService`: Creación, actualización, baja lógica y carga de portadas de libros (**HU-03**, **HU-04**).
  - `MemberService`: Registro y consulta de socios lectores (**HU-05**).
  - `LoanService`: Valida límites de préstamo, mora activa, descuenta inventario y procesa devoluciones (**HU-06**, **HU-07**, **HU-09**).
  - `ReportService`: Genera el reporte inmutable de los libros con mayor volumen de circulación (**HU-10**).

### 2.3 Capa de Adaptadores e Infraestructura (`app/Adapters/`)
Implementa las tecnologías concretas sobre las interfaces de aplicación:
- **Persistencia PostgreSQL (`app/Adapters/Persistence/`):** Implementaciones usando sentencias preparadas de PDO sobre el esquema `library`. Aplica bloqueo pesimista `SELECT ... FOR UPDATE` o control optimista para evitar sobre-préstamos concurrentes (**RNF-2**).
- **Seguridad JWT (`app/Adapters/Security/JwtTokenIssuer.php`):** Generación de firmas HMAC-SHA256 con payload estándar conteniendo `sub` (UUID del usuario), `role`, `iat` y `exp` fijado en 28,800 segundos (8 horas).
- **Composición IoC (`app/Providers/AppServiceProvider.php`):** Registra las 13 asociaciones entre puertos e implementaciones conforme al **Artículo III**.

---

## 3. Capa de Transporte HTTP en Laravel

La API se publica bajo el prefijo unificado `/api/v1` en `routes/api.php` cumpliendo los 21 endpoints del contrato formal:

- **Autenticación:**
  - `POST /api/v1/auth/login` (HU-01, CA-01.1, CA-01.2)
- **Usuarios y Bibliotecarios:**
  - `POST /api/v1/users` (HU-02, CA-02.1, CA-02.2)
- **Categorías:**
  - `GET /api/v1/categories` (T-02, T-08)
- **Libros:**
  - `GET /api/v1/books` (HU-03, CA-03.1)
  - `POST /api/v1/books` (HU-03, CA-03.2)
  - `GET /api/v1/books/{id}` (HU-03)
  - `PUT /api/v1/books/{id}` (HU-03)
  - `DELETE /api/v1/books/{id}` (HU-04, CA-04.1)
  - `POST /api/v1/books/{id}/cover` (HU-03, T-15)
- **Socios (Miembros):**
  - `GET /api/v1/members` (HU-05, CA-05.1)
  - `POST /api/v1/members` (HU-05, CA-05.2)
  - `GET /api/v1/members/{id}` (HU-05)
- **Préstamos:**
  - `POST /api/v1/loans` (HU-06, CA-06.1, CA-06.2)
  - `GET /api/v1/loans` (HU-06)
  - `GET /api/v1/loans/overdue` (HU-09, CA-09.1)
  - `GET /api/v1/loans/{id}` (HU-06)
  - `POST /api/v1/loans/{loanId}/items/{itemId}/return` (HU-07, CA-07.1, CA-07.2)
- **Reportes:**
  - `GET /api/v1/reports/most-borrowed` (HU-10, CA-10.1, CA-10.2)
- **Salud del Sistema:**
  - `GET /api/v1/health` (T-01, RNF-5)

### Manejo Unificado de Errores
Todo error de dominio o validación HTTP se transforma en el cuerpo estándar JSON definido en `api-contract.md` §2:
```json
{
  "code": "BOOK_OUT_OF_STOCK",
  "message": "No hay ejemplares disponibles para el libro solicitado.",
  "details": []
}
```
Se contemplan los 14 códigos canónicos de error: `INVALID_CREDENTIALS`, `USERNAME_NOT_UNIQUE`, `ISBN_NOT_UNIQUE`, `BOOK_NOT_FOUND`, `BOOK_OUT_OF_STOCK`, `MEMBER_NOT_FOUND`, `MEMBER_DOCUMENT_NOT_UNIQUE`, `MEMBER_LIMIT_EXCEEDED`, `MEMBER_HAS_OVERDUE_LOANS`, `LOAN_NOT_FOUND`, `ITEM_ALREADY_RETURNED`, `UNAUTHORIZED`, `FORBIDDEN`, `VALIDATION_ERROR`.

---

## 4. Frontend React SPA (`app`)

El frontend está desarrollado con **React** y empaquetado mediante **Vite**, desplegado en un servidor web ultraligero **Nginx** en contenedor Docker (puerto `5173`).

### 4.1 Módulos y Pantallas de Usuario (T-22)
1. **Pantalla de Login:** Formulario de autenticación que captura el token Bearer JWT y almacena el rol (`admin` o `librarian`) en el estado global de React.
2. **Catálogo de Libros:** Vista principal con tabla interactiva, paginación, búsqueda instantánea por título o autor, filtro dinámico por las 5 categorías, visualización de existencias (`available_copies / total_copies`) y acciones de alta, edición y baja lógica.
3. **Gestión de Socios:** Directorio de socios lectores con buscador por documento de identidad y formulario de registro.
4. **Registro de Préstamos:** Flujo de creación de préstamo con selección de socio, validación en tiempo real del límite de 3 libros y alerta inmediata si el socio presenta suspensiones o moras.
5. **Recepción de Devoluciones:** Listado de préstamos activos con botón de devolución unitaria por línea, actualizando al instante el stock del catálogo.
6. **Monitor de Préstamos Vencidos:** Pestaña especializada que lista las líneas con fecha de retorno expirada (`due_at < now`).
7. **Reporte de Circulación:** Visualización del ranking Top 10 de libros más solicitados con métricas agregadas.

---

## 5. Infraestructura y Despliegue (`infra`)

La infraestructura se define en `docker-compose.yml` (T-20) conectando tres servicios sobre la red privada `library-net`:

1. **`db` (PostgreSQL 16 Alpine):** Servicio de base de datos con volumen persistente `db-data`, arranque en frío con esquema vacío y healthcheck con `pg_isready`.
2. **`api` (PHP Laravel):** Contenedor con PHP 8.2-cli y servidor integrado de Laravel (`php artisan serve`), montado sobre el puerto `8000`. Ejecuta las migraciones de Artisan en su punto de entrada (`entrypoint.sh`).
3. **`app` (React con Vite y Nginx):** Construcción multi-etapa que compila los componentes de React con Vite y sirve los archivos estáticos a través de Nginx en el puerto `5173`.

---

## 6. Utilidad de Siembra Automatizada (`tool`)

El sembrador `seed.php` (T-21) actúa como cliente externo de la API REST para poblar el sistema con el conjunto mínimo de datos de demostración requerido por la especificación:
- 1 usuario administrador inicial autenticado.
- 1 usuario bibliotecario (`ana`).
- 40 libros distribuidos en las 5 categorías.
- 10 socios lectores con documentos válidos.
- 15 préstamos con variaciones de estado: préstamos abiertos, devoluciones parciales y 3 préstamos en estado vencido utilizando la cabecera `X-Simulated-Time`.

---

## 7. Mapeo Detallado de Entidades y Objetos de Valor

### 7.1 Objeto de Valor `Isbn`
Implementado en `app/Domain/Isbn.php`. Representa el International Standard Book Number en su versión de 13 dígitos numéricos:
- Inmutable por diseño (`readonly class`).
- Validación en el constructor: expresión regular `^\d{13}$` y cálculo del dígito de verificación ISBN-13 (ponderaciones alternadas 1 y 3 módulo 10).
- Métodos de acceso: `value(): string`, `__toString(): string`.
- Garantiza la regla innegociable **RN-4** antes de cualquier persistencia.

### 7.2 Objeto de Valor `Quantity`
Implementado en `app/Domain/Quantity.php`:
- Representa el volumen de copias en existencia física.
- Constructor que restringe valores estrictamente mayores o iguales a cero (`$value >= 0`).
- Operaciones aritméticas inmutables: `add(Quantity $other): Quantity`, `subtract(Quantity $other): Quantity`.
- Lanza `DomainError` con código `VALIDATION_ERROR` si se intenta instanciar con enteros negativos.

### 7.3 Objeto de Valor `DateRange`
Implementado en `app/Domain/DateRange.php`:
- Encapsula un intervalo temporal con fecha de inicio y fecha de finalización.
- Garantiza que `start <= end`.
- Método de cálculo `daysDifference(): int` para computar plazos de préstamo y días de mora acumulados.

### 7.4 Agregado de Dominio `Book`
Implementado en `app/Domain/Book.php`:
- Propiedades: `id` (string UUID), `title`, `author`, `isbn` (Isbn), `categoryId` (string UUID), `totalCopies` (Quantity), `availableCopies` (Quantity), `coverKey` (?string), `deletedAt` (?DateTimeImmutable).
- Invariante de stock: en todo momento se verifica que `availableCopies <= totalCopies` (RN-1).
- Método `withdraw(): Book`:
  - Valida que `availableCopies > 0`. Si es 0, lanza excepción de dominio con código `BOOK_OUT_OF_STOCK` (CA-06.1).
  - Retorna una nueva instancia inmutable con `availableCopies` decrementado en 1.
- Método `restock(): Book`:
  - Valida que `availableCopies < totalCopies`. Si es igual, no permite incrementar por encima del inventario total.
  - Retorna una nueva instancia con `availableCopies` incrementado en 1.
- Método `markDeleted(DateTimeImmutable $now): Book`:
  - Fija `deletedAt` con la marca de tiempo actual (baja lógica).
  - Preserva el historial transaccional en la base de datos (ADR-003, T-04).

### 7.5 Agregado de Dominio `Loan` y `LoanItem`
Implementados en `app/Domain/Loan.php` y `app/Domain/LoanItem.php`:
- `Loan`:
  - Propiedades: `id` (UUID), `memberId` (UUID), `loanedByUserId` (UUID), `loanedAt` (DateTimeImmutable), `items` (array de LoanItem).
  - Regla **RN-3**: el constructor valida que el arreglo de ítems contenga entre 1 y 3 elementos (`count($items) >= 1 && count($items) <= 3`). Si la lista está vacía o supera 3, lanza `MEMBER_LIMIT_EXCEEDED`.
  - Regla **RN-2**: cada ítem se genera con una fecha límite de devolución `dueAt = loanedAt + 14 días`.
- `LoanItem`:
  - Propiedades: `id` (UUID), `loanId` (UUID), `bookId` (UUID), `bookTitle` (string), `isbn` (string), `categoryName` (string), `dueAt` (DateTimeImmutable), `returnedAt` (?DateTimeImmutable).
  - Congelamiento histórico (ADR-004): `bookTitle`, `isbn` y `categoryName` capturan el valor estático al momento de originar el préstamo.
  - Método `markReturned(DateTimeImmutable $now): LoanItem`:
    - Si `returnedAt` ya tiene un valor no nulo, lanza `ITEM_ALREADY_RETURNED` (CA-07.2).
    - Asigna `returnedAt = $now`.
  - Método `isOverdue(DateTimeImmutable $currentDate): bool`:
    - Retorna `true` si `returnedAt === null && $currentDate > dueAt`.

---

## 8. Flujos de Secuencia de Casos de Uso

### 8.1 Flujo de Préstamo de Libros (HU-06)
1. El bibliotecario selecciona el socio y entre 1 y 3 libros disponibles en el frontend React.
2. React envía `POST /api/v1/loans` con token Bearer en cabecera HTTP.
3. El middleware `AuthenticateWithJwt` valida la firma del token y extrae el identificador del usuario (`sub`).
4. `LibraryApiController` recibe el DTO de entrada y delega la ejecución a `LoanService::lendBooks()`.
5. `LoanService` invoca `MemberRepository::findById()`:
   - Si no existe, lanza `MEMBER_NOT_FOUND`.
6. `LoanService` invoca `LoanReader::countActiveLoansByMember()`:
   - Si el socio ya tiene préstamos activos que sumados a la nueva solicitud superan 3, lanza `MEMBER_LIMIT_EXCEEDED` (CA-06.2).
7. `LoanService` invoca `LoanReader::hasOverdueLoans()`:
   - Si el socio posee alguna línea con `due_at < now` y `returned_at IS NULL`, lanza `MEMBER_HAS_OVERDUE_LOANS` (CA-06.1).
8. `DbTransactionRunner` inicia una transacción atómica en PostgreSQL (`BEGIN`):
   - Itera los libros solicitados ejecutando `BookRepository::lockAndFindById()` (`SELECT ... FOR UPDATE`).
   - Verifica que cada libro no esté dado de baja y tenga `available_copies > 0`. Si no hay existencias, lanza `BOOK_OUT_OF_STOCK`.
   - Ejecuta `Book::withdraw()` decrementando `available_copies`.
   - Persiste el libro actualizado mediante `BookRepository::save()`.
   - Crea la cabecera `Loan` y las líneas `LoanItem` con `due_at = now + 14 días`.
   - Persiste la transacción en `LoanRepository::save()`.
   - Ejecuta `COMMIT`.
9. `LibraryApiController` responde con código HTTP `201 Created` y el JSON estructurado del préstamo.

### 8.2 Flujo de Devolución de Ejemplares (HU-07)
1. El bibliotecario visualiza la lista de préstamos activos en React y hace clic en "Devolver" sobre un ejemplar.
2. React envía `POST /api/v1/loans/{loanId}/items/{itemId}/return`.
3. `AuthenticateWithJwt` autentica al operador.
4. `LoanService::returnBookItem($loanId, $itemId)`:
   - Inicia transacción en `DbTransactionRunner`.
   - Carga el préstamo y localiza la línea solicitada.
   - Si la línea ya fue devuelta (`returned_at IS NOT NULL`), lanza `ITEM_ALREADY_RETURNED` con HTTP 409.
   - Marca `returned_at = now` en la línea.
   - Carga el libro asociado y ejecuta `Book::restock()` incrementando `available_copies` en 1.
   - Guarda los cambios en `LoanRepository` y `BookRepository`.
   - Ejecuta `COMMIT`.
5. Retorna HTTP `200 OK` con el detalle actualizado. En React, el estado local actualiza inmediatamente el stock del libro en el catálogo.

### 8.3 Flujo de Monitoreo de Préstamos Vencidos (HU-09)
1. En React, la pestaña "Vencidos" solicita `GET /api/v1/loans/overdue`.
2. `LoanService::getOverdueLoans()` consulta `LoanReader::findOverdueItems()`.
3. `PostgresLoanReader` ejecuta la consulta SQL:
   ```sql
   SELECT li.id, li.loan_id, li.book_id, li.book_title, li.isbn, li.category_name,
          li.due_at, l.member_id, m.full_name AS member_name, m.document_number,
          EXTRACT(DAY FROM (NOW() - li.due_at))::int AS days_overdue
   FROM library.loan_item li
   JOIN library.loan l ON l.id = li.loan_id
   JOIN library.member m ON m.id = l.member_id
   WHERE li.returned_at IS NULL
     AND li.due_at < NOW()
   ORDER BY li.due_at ASC;
   ```
4. Se serializa la lista en JSON y React renderiza las alertas de mora con los días de retraso calculados.

---

## 9. Seguridad y Políticas de Acceso (RBAC)

El sistema define dos roles con responsabilidades diferenciadas según la especificación:

| Funcionalidad / Endpoint | Rol Administrador (`admin`) | Rol Bibliotecario (`librarian`) |
|---|:---:|:---:|
| Iniciar sesión (`POST /auth/login`) | Permitido | Permitido |
| Crear bibliotecarios (`POST /users`) | **Permitido** | Denegado (HTTP 403 `FORBIDDEN`) |
| Listar catálogo de libros (`GET /books`) | Permitido | Permitido |
| Crear nuevo libro (`POST /books`) | Permitido | Permitido |
| Modificar libro (`PUT /books/{id}`) | Permitido | Permitido |
| Baja lógica de libro (`DELETE /books/{id}`) | **Permitido** | Denegado (HTTP 403 `FORBIDDEN`) |
| Subir portada de libro (`POST /books/{id}/cover`) | Permitido | Permitido |
| Listar socios (`GET /members`) | Permitido | Permitido |
| Registrar socio (`POST /members`) | Permitido | Permitido |
| Registrar préstamo (`POST /loans`) | Permitido | Permitido |
| Devolver libro (`POST /loans/.../return`) | Permitido | Permitido |
| Ver préstamos vencidos (`GET /loans/overdue`) | Permitido | Permitido |
| Reporte Top 10 (`GET /reports/most-borrowed`) | Permitido | Permitido |
