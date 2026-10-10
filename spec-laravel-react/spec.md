# Especificación de Requerimientos — Manager Library (Laravel + React)

Especificación funcional completa del sistema **Manager Library** adaptado a **PHP Laravel** en el backend y **React con Vite** en el frontend, estructurado según la metodología **Spec-Driven Development (SDD)**.

---

## 1. Actores del Sistema

- **Administrador (`admin`):** Responsable de la gestión del personal bibliotecario, configuración general del sistema y baja lógica de ejemplares obsoletos o deteriorados. Posee acceso total a todos los módulos.
- **Bibliotecario (`librarian`):** Operador de sala responsable de la atención al público, catalogación de libros, registro de socios lectores, formalización de préstamos y recepción de devoluciones.
- **Socio Lector (Miembro):** Usuario final que disfruta del préstamo de libros físicos. Identificado por su documento de identidad único; no posee credenciales de acceso directo al sistema (operado por el bibliotecario).

---

## 2. Historias de Usuario y Criterios de Aceptación

### HU-01: Autenticación de Usuarios
**Como** personal de la biblioteca (Administrador o Bibliotecario),  
**Quiero** autenticarme en el sistema con mi usuario y contraseña,  
**Para** acceder a las funcionalidades del sistema según mis permisos asignados.

- **Criterio de Aceptación CA-01.1:** Al ingresar credenciales correctas, el sistema responde HTTP 200 con un token de acceso Bearer JWT, indicando el rol (`admin` o `librarian`) y con vigencia de 8 horas (RN-5).
- **Criterio de Aceptación CA-01.2:** Si el usuario no existe o la contraseña no coincide, el sistema responde HTTP 401 con el código canónico de error `INVALID_CREDENTIALS` sin indicar cuál de los dos datos falló.

---

### HU-02: Creación de Bibliotecarios
**Como** Administrador del sistema,  
**Quiero** registrar nuevos usuarios con rol de bibliotecario,  
**Para** habilitar al personal en la operación de préstamos y catálogo.

- **Criterio de Aceptación CA-02.1:** Un administrador autenticado envía `POST /api/v1/users` con `username`, `password` y `role: librarian`. El sistema normaliza el nombre de usuario (sin espacios, minúsculas), almacena la contraseña hasheada y responde HTTP 201 con el registro creado.
- **Criterio de Aceptación CA-02.2:** Si el `username` ya está en uso por otro usuario, el sistema responde HTTP 409 con el código de error `USERNAME_NOT_UNIQUE`.
- **Criterio de Aceptación CA-02.3:** Si un bibliotecario intenta invocar este endpoint, el sistema responde HTTP 403 `FORBIDDEN`.

---

### HU-03: Gestión del Catálogo de Libros
**Como** Bibliotecario o Administrador,  
**Quiero** registrar y mantener actualizado el catálogo bibliográfico,  
**Para** ofrecer información certera sobre el inventario disponible.

- **Criterio de Aceptación CA-03.1:** El listado `GET /api/v1/books` permite buscar por texto (`q`) en título o autor, filtrar por `category_id` y navegar resultados con paginación (`page`, `per_page`), retornando únicamente libros no eliminados (`deleted_at IS NULL`).
- **Criterio de Aceptación CA-03.2:** Al crear un libro (`POST /api/v1/books`), el ISBN debe cumplir el formato canónico de 13 dígitos numéricos (RN-4). Si ya existe un libro activo con el mismo ISBN, el sistema responde HTTP 409 `ISBN_NOT_UNIQUE`. El libro nace con `available_copies = total_copies`.
- **Criterio de Aceptación CA-03.3:** La actualización `PUT /api/v1/books/{id}` permite ajustar título, autor, categoría y cantidad total de copias, garantizando que el total nunca sea inferior a las copias prestadas actualmente (`available_copies >= 0`).

---

### HU-04: Baja Lógica de Libros
**Como** Administrador,  
**Quiero** dar de baja libros del catálogo,  
**Para** retirarlos de circulación sin destruir el historial de préstamos pasados.

- **Criterio de Aceptación CA-04.1:** Al invocar `DELETE /api/v1/books/{id}`, el sistema establece la fecha y hora actual en la columna `deleted_at` sin borrar la fila física en PostgreSQL (ADR-003). Responde HTTP 204 No Content.
- **Criterio de Aceptación CA-04.2:** El libro dado de baja desaparece inmediatamente de las búsquedas del catálogo y de la lista de selección para nuevos préstamos.
- **Criterio de Aceptación CA-04.3:** Si el usuario no tiene rol de administrador, la operación es denegada con HTTP 403 `FORBIDDEN`.

---

### HU-05: Registro y Consulta de Socios Lectores
**Como** Bibliotecario,  
**Quiero** registrar y consultar a los socios de la biblioteca,  
**Para** gestionar quién tiene derecho al préstamo de ejemplares físicos.

- **Criterio de Aceptación CA-05.1:** La consulta `GET /api/v1/members` permite filtrar por número de documento de identidad o nombre completo.
- **Criterio de Aceptación CA-05.2:** Al crear un socio (`POST /api/v1/members`), el `document_number` debe ser único en la tabla `library.member`. Si ya existe, responde HTTP 409 `MEMBER_DOCUMENT_NOT_UNIQUE`.

---

### HU-06: Creación de Préstamos
**Como** Bibliotecario,  
**Quiero** registrar un nuevo préstamo de libros a un socio,  
**Para** autorizar la salida temporal de ejemplares del acervo físico.

- **Criterio de Aceptación CA-06.1:** Si el socio tiene al menos una línea de préstamo previa vencida (`due_at < now` y `returned_at IS NULL`), el sistema rechaza el préstamo con HTTP 409 y código `MEMBER_HAS_OVERDUE_LOANS`.
- **Criterio de Aceptación CA-06.2:** Un socio puede tener un máximo de 3 libros en préstamo activo de forma concurrente (RN-3). Si el socio ya tiene 2 libros y solicita 2 más, o si solicita más de 3 en una sola petición, el sistema rechaza la operación con HTTP 409 y código `MEMBER_LIMIT_EXCEEDED`.
- **Criterio de Aceptación CA-06.3:** Si alguno de los libros solicitados no posee ejemplares disponibles (`available_copies = 0`), el sistema rechaza la operación completa de manera atómica con HTTP 409 y código `BOOK_OUT_OF_STOCK` (RN-1).
- **Criterio de Aceptación CA-06.4:** Cada libro prestado recibe automáticamente una fecha límite de devolución fijada en exactamente 14 días calendario a partir del momento del préstamo (`due_at = loaned_at + 14 días`, RN-2).
- **Criterio de Aceptación CA-06.5:** Cada línea de préstamo almacena una copia congelada e inmutable de `book_title`, `isbn` y `category_name` en el instante del préstamo (ADR-004, Artículo VIII).

---

### HU-07: Devolución de Libros
**Como** Bibliotecario,  
**Quiero** registrar la devolución de un ejemplar prestado,  
**Para** reincorporarlo al inventario disponible para otros usuarios.

- **Criterio de Aceptación CA-07.1:** Al procesar la devolución de una línea (`POST /api/v1/loans/{loanId}/items/{itemId}/return`), el sistema asigna la fecha y hora actual a `returned_at` e incrementa en 1 la columna `available_copies` del libro correspondiente. Responde HTTP 200 OK.
- **Criterio de Aceptación CA-07.2:** Si la línea de préstamo ya había sido devuelta previamente (`returned_at IS NOT NULL`), el sistema responde HTTP 409 con el código de error `ITEM_ALREADY_RETURNED` sin alterar el stock.

---

### HU-08: Carga de Portadas de Libros
**Como** Bibliotecario,  
**Quiero** subir una imagen de portada para un libro del catálogo,  
**Para** mejorar la presentación visual en el frontend.

- **Criterio de Aceptación CA-08.1:** Se admite la subida de archivos PNG o JPG de hasta 2 megabytes en el endpoint `POST /api/v1/books/{id}/cover`. El archivo se guarda en el adaptador de disco (`storage/app/covers/`) y se actualiza `cover_key`.
- **Criterio de Aceptación CA-08.2:** Si el archivo supera 2 MB o no corresponde a una imagen válida, responde HTTP 422 `VALIDATION_ERROR`.

---

### HU-09: Monitoreo de Préstamos Vencidos
**Como** Bibliotecario,  
**Quiero** consultar los préstamos que han superado su fecha límite de devolución,  
**Para** realizar seguimiento y contactar a los socios con retrasos.

- **Criterio de Aceptación CA-09.1:** El endpoint `GET /api/v1/loans/overdue` retorna todas las líneas activas no devueltas (`returned_at IS NULL`) cuya fecha `due_at` sea estrictamente menor a la fecha actual, incluyendo el cálculo de días de mora transcurridos.

---

### HU-10: Reporte de Libros Más Prestados
**Como** Administrador o Bibliotecario,  
**Quiero** consultar el reporte de los libros con mayor volumen de préstamos,  
**Para** tomar decisiones sobre adquisición de nuevos ejemplares y rotación.

- **Criterio de Aceptación CA-10.1:** El endpoint `GET /api/v1/reports/most-borrowed` agrupa las líneas históricas de `library.loan_item` y devuelve el Top 10 ordenado de mayor a menor cantidad de préstamos.
- **Criterio de Aceptación CA-10.2:** El reporte utiliza exclusivamente los datos congelados (`book_title`, `isbn`, `category_name`) registrados en `loan_item`, garantizando que la modificación posterior de un título en el catálogo vivo no altere la consistencia contable del reporte histórico (ADR-004).

---

## 3. Reglas de Negocio Innegociables (RN-1 a RN-5)

1. **RN-1 (Consistencia de Inventario):** En todo momento, la cantidad de ejemplares disponibles debe respetar la relación `0 <= available_copies <= total_copies`. Garantizada por código de dominio y por la restricción física `ck_book_available_range` en PostgreSQL.
2. **RN-2 (Plazo de Préstamo):** Todo préstamo se otorga por un plazo inmutable de exactamente 14 días calendario desde su expedición.
3. **RN-3 (Cupo Máximo de Préstamo):** Un socio lector puede tener prestados como máximo 3 libros en simultáneo. Cada solicitud de préstamo admite entre 1 y 3 libros.
4. **RN-4 (Estructura de ISBN):** Todo código ISBN debe ser una cadena numérica canónica de exactamente 13 dígitos con algoritmo de dígito de verificación válido.
5. **RN-5 (Duración de Sesión):** Los tokens JWT emitidos en la autenticación expiran tras 8 horas continuas (28,800 segundos), tras lo cual se exige reautenticación.

---

## 4. Requerimientos No Funcionales (RNF-1 a RNF-5)

1. **RNF-1 (Sembrado Automatizado por API):** La herramienta de siembra (`tool`) debe interactuar con el sistema exclusivamente mediante peticiones HTTP RESTful contra `/api/v1`, sin inyección SQL directa.
2. **RNF-2 (Control de Concurrencia Transaccional):** Ante solicitudes simultáneas que compitan por el último ejemplar disponible de un libro, el motor relacional y la capa de aplicación deben garantizar que solo una operación sea exitosa (HTTP 201) y la restante reciba HTTP 409 `BOOK_OUT_OF_STOCK`.
3. **RNF-3 (Seguridad y Confidencialidad):** Ningún secreto (contraseñas, claves privadas, hashes) puede figurar en repositorios de código, respuestas HTTP públicas ni archivos de registro (logs).
4. **RNF-4 (Desacoplamiento Tecnológico):** El dominio de negocio debe permanecer independiente de frameworks y bibliotecas de terceros (Arquitectura Hexagonal).
5. **RNF-5 (Portabilidad y Despliegue en Contenedores):** El sistema completo debe iniciar de forma reproducible mediante Docker Compose en un solo comando (`docker compose up -d`), con base de datos limpia poblada por migraciones.
