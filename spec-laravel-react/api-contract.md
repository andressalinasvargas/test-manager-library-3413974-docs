# Contrato de la API REST — Manager Library (Laravel API /api/v1)

Especificación completa del contrato HTTP de la API REST implementada en **PHP con Laravel** (`api`) y consumida por el cliente **React SPA** (`app`) y la herramienta **Seeder** (`tool`).

Todos los endpoints se exponen obligatoriamente bajo el prefijo unificado `/api/v1` en `routes/api.php`.

---

## 1. Convenciones Globales

1. **Protocolo y Formato:** Todas las peticiones y respuestas intercambian cargas en `application/json` codificadas en UTF-8.
2. **Autenticación (RN-5):** Los endpoints protegidos exigen la cabecera HTTP:
   ```http
   Authorization: Bearer <token_jwt>
   ```
   El token es firmado mediante algoritmo HMAC-SHA256 (`JWT_SECRET`) y posee una duración de exactamente 8 horas (28,800 segundos).
3. **Cuerpo Canónico de Error:** Todo fallo HTTP (4xx o 5xx) responde obligatoriamente con la estructura JSON uniforme:
   ```json
   {
     "code": "CODIGO_ERROR",
     "message": "Descripción comprensible para el usuario en español",
     "details": []
   }
   ```

---

## 2. Catálogo de Códigos de Error (14 Códigos Oficiales)

| Código de Error | Código HTTP | Descripción y Regla Asociada |
|---|:---:|---|
| `INVALID_CREDENTIALS` | 401 | Credenciales de acceso incorrectas o token ausente/expirado (CA-01.2). |
| `UNAUTHORIZED` | 401 | Se requiere cabecera Bearer válida para acceder al recurso. |
| `FORBIDDEN` | 403 | El usuario autenticado carece del rol necesario (ej. bibliotecario intentando crear usuarios o borrar libros). |
| `USERNAME_NOT_UNIQUE` | 409 | Ya existe un usuario registrado con el nombre de usuario indicado (CA-02.2). |
| `ISBN_NOT_UNIQUE` | 409 | Ya existe un libro activo registrado con ese código ISBN de 13 dígitos. |
| `BOOK_NOT_FOUND` | 404 | El identificador UUID del libro no existe o fue dado de baja lógica. |
| `BOOK_OUT_OF_STOCK` | 409 | El libro solicitado tiene 0 ejemplares disponibles (`available_copies = 0`). |
| `MEMBER_NOT_FOUND` | 404 | El socio no existe en el sistema. |
| `MEMBER_DOCUMENT_NOT_UNIQUE` | 409 | Ya existe un socio con ese número de documento de identidad. |
| `MEMBER_LIMIT_EXCEEDED` | 409 | El socio ya posee 3 libros en préstamo activo o la operación excede el cupo (CA-06.2, RN-3). |
| `MEMBER_HAS_OVERDUE_LOANS` | 409 | El socio presenta préstamos con fecha límite vencida y está inhabilitado (CA-06.1). |
| `LOAN_NOT_FOUND` | 404 | El identificador UUID del préstamo no existe. |
| `ITEM_ALREADY_RETURNED` | 409 | La línea de préstamo solicitada ya fue devuelta previamente (CA-07.2). |
| `VALIDATION_ERROR` | 422 | Error en la validación semántica de campos de entrada (formato de ISBN, cantidades negativas, etc.). |

---

## 3. Catálogo de los 21 Endpoints de la API

### 3.1 Salud del Sistema
- **`GET /api/v1/health` (T-01, RNF-5):**
  - **Acceso:** Público.
  - **Respuesta 200:** `{"status":"ok"}`.

### 3.2 Autenticación
- **`POST /api/v1/auth/login` (HU-01, CA-01.1, CA-01.2):**
  - **Acceso:** Público.
  - **Body:** `{"username": "admin", "password": "..."}`.
  - **Respuesta 200:** `{"access_token": "...", "token_type": "Bearer", "expires_in": 28800, "user_id": "...", "role": "admin"}`.
  - **Respuesta 401:** Error `INVALID_CREDENTIALS`.

### 3.3 Gestión de Usuarios (Personal)
- **`POST /api/v1/users` (HU-02, CA-02.1, CA-02.2):**
  - **Acceso:** Solo Administrador (`role: admin`).
  - **Body:** `{"username": "ana", "password": "...", "role": "librarian"}`.
  - **Respuesta 201:** `{"id": "...", "username": "ana", "role": "librarian"}`.
  - **Respuesta 409:** Error `USERNAME_NOT_UNIQUE`.

### 3.4 Categorías
- **`GET /api/v1/categories` (T-02, T-08):**
  - **Acceso:** Autenticado (Admin o Bibliotecario).
  - **Respuesta 200:** Lista de las 5 categorías inmutables `[{"id": "...", "name": "Novela"}, ...]`.

### 3.5 Libros y Portadas
- **`GET /api/v1/books` (HU-03, CA-03.1):**
  - **Acceso:** Autenticado.
  - **Query Params:** `category_id`, `q` (búsqueda por título o autor), `page`, `per_page`.
  - **Respuesta 200:** `{"items": [...], "total": 40, "page": 1, "per_page": 20, "total_pages": 2}`.
- **`POST /api/v1/books` (HU-03, CA-03.2):**
  - **Acceso:** Autenticado.
  - **Body:** `{"title": "...", "author": "...", "isbn": "9781234567890", "category_id": "...", "total_copies": 5}`.
  - **Respuesta 201:** Libro creado con `available_copies = total_copies`.
  - **Respuesta 409:** Error `ISBN_NOT_UNIQUE`.
- **`GET /api/v1/books/{id}` (HU-03):**
  - **Acceso:** Autenticado.
  - **Respuesta 200:** Detalle del libro.
  - **Respuesta 404:** Error `BOOK_NOT_FOUND`.
- **`PUT /api/v1/books/{id}` (HU-03):**
  - **Acceso:** Autenticado.
  - **Body:** Modificación de `title`, `author`, `category_id`, `total_copies`.
  - **Respuesta 200:** Libro actualizado.
- **`DELETE /api/v1/books/{id}` (HU-04, CA-04.1):**
  - **Acceso:** Solo Administrador (`role: admin`).
  - **Respuesta 204:** Baja lógica exitosa (`deleted_at = now`).
  - **Respuesta 403:** Error `FORBIDDEN` si intenta un bibliotecario.
- **`POST /api/v1/books/{id}/cover` (HU-03, T-15):**
  - **Acceso:** Autenticado.
  - **Body:** Archivo `multipart/form-data` con campo `cover` (máx. 2 MB, PNG o JPG).
  - **Respuesta 200:** `{"cover_key": "..."}`.

### 3.6 Socios Lectores
- **`GET /api/v1/members` (HU-05, CA-05.1):**
  - **Acceso:** Autenticado.
  - **Query Params:** `q` (por documento o nombre), `page`, `per_page`.
  - **Respuesta 200:** Paginación de socios.
- **`POST /api/v1/members` (HU-05, CA-05.2):**
  - **Acceso:** Autenticado.
  - **Body:** `{"full_name": "...", "document_number": "..."}`.
  - **Respuesta 201:** Socio creado.
  - **Respuesta 409:** Error `MEMBER_DOCUMENT_NOT_UNIQUE`.
- **`GET /api/v1/members/{id}` (HU-05):**
  - **Acceso:** Autenticado.
  - **Respuesta 200:** Detalle del socio.

### 3.7 Préstamos y Devoluciones
- **`POST /api/v1/loans` (HU-06, CA-06.1, CA-06.2):**
  - **Acceso:** Autenticado.
  - **Headers Opcionales:** `X-Simulated-Time` para pruebas y simulaciones de tiempo.
  - **Body:** `{"member_id": "...", "items": [{"book_id": "..."}, ...]}` (de 1 a 3 libros).
  - **Respuesta 201:** Préstamo creado con fecha límite `due_at = loaned_at + 14 días` y descuento de copias disponibles.
  - **Respuesta 409:** Errores `MEMBER_HAS_OVERDUE_LOANS`, `MEMBER_LIMIT_EXCEEDED` o `BOOK_OUT_OF_STOCK`.
- **`GET /api/v1/loans` (HU-06):**
  - **Acceso:** Autenticado.
  - **Respuesta 200:** Lista de préstamos con estado calculado (`open`, `overdue`, `closed`).
- **`GET /api/v1/loans/overdue` (HU-09, CA-09.1):**
  - **Acceso:** Autenticado.
  - **Respuesta 200:** Lista de líneas de préstamo con `returned_at IS NULL AND due_at < now`.
- **`GET /api/v1/loans/{id}` (HU-06):**
  - **Acceso:** Autenticado.
  - **Respuesta 200:** Detalle del préstamo con sus líneas e historial.
- **`POST /api/v1/loans/{loanId}/items/{itemId}/return` (HU-07, CA-07.1, CA-07.2):**
  - **Acceso:** Autenticado.
  - **Respuesta 200:** Devolución procesada (`returned_at = now`) y copias disponibles incrementadas en 1.
  - **Respuesta 409:** Error `ITEM_ALREADY_RETURNED` si ya estaba devuelta.

### 3.8 Reportes Analíticos
- **`GET /api/v1/reports/most-borrowed` (HU-10, CA-10.1, CA-10.2):**
  - **Acceso:** Autenticado.
  - **Respuesta 200:** Ranking Top 10 de libros con mayor cantidad de préstamos, computado sobre las instantáneas congeladas de `loan_item` (ADR-004).
