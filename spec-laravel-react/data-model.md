# Modelo de Datos — Manager Library (PostgreSQL 16 en Laravel)

Especificación formal del modelo de datos relacional para **Manager Library** implementado mediante migraciones de **Laravel** (`php artisan migrate`) sobre **PostgreSQL 16**.

Todo el DDL reside exclusivamente en la migración de Laravel dentro del repositorio `api` (`database/migrations/2026_10_10_000001_create_library_schema.php`), respetando el principio de base de datos vacía al inicio (ADR-001, Artículo V).

---

## 1. Esquema Relacional

Todas las estructuras de persistencia pertenecen al esquema dedicado `library`. No se crea ninguna tabla en el esquema por defecto `public`.

```sql
CREATE SCHEMA IF NOT EXISTS library;
```

---

## 2. Definición de Tablas (Entidades Físicas)

El modelo está compuesto por **seis tablas en singular**, sin tablas auxiliares de frameworks (como las que agregan sesiones, colas o tokens personales en Laravel) y sin columnas innecesarias de auditoría como `created_at` o `updated_at` (§8 del modelo de datos):

### 2.1 Tabla `library.category`
Catálogo inmutable de géneros literarios sembrados al iniciar el sistema.
- `id` (uuid, NOT NULL, PRIMARY KEY)
- `name` (varchar(120), NOT NULL, UNIQUE)

### 2.2 Tabla `library.book`
Registro de títulos bibliográficos y existencias físicas.
- `id` (uuid, NOT NULL, PRIMARY KEY)
- `title` (varchar(250), NOT NULL)
- `author` (varchar(200), NOT NULL)
- `isbn` (char(13), NOT NULL) — 13 dígitos numéricos canónicos.
- `category_id` (uuid, NOT NULL, FK hacia `library.category(id)`)
- `total_copies` (integer, NOT NULL) — Ejemplares totales en acervo.
- `available_copies` (integer, NOT NULL) — Ejemplares disponibles para préstamo inmediato.
- `cover_key` (varchar(512), NULL) — Ruta relativa al archivo de portada en disco.
- `deleted_at` (timestamptz, NULL) — Marca de baja lógica (preserva el historial).

**Restricciones e Índices:**
- Clave foránea **FK-1**: `fk_book_category` hacia `category(id)` con `ON DELETE RESTRICT`.
- Restricción de chequeo **CK-1** (ADR-002, RN-1): `CONSTRAINT ck_book_available_range CHECK (available_copies >= 0 AND available_copies <= total_copies)`.
- Índice de unicidad parcial **UX-1** (ADR-003): `CREATE UNIQUE INDEX ux_book_isbn_active ON library.book (isbn) WHERE deleted_at IS NULL;`.
- Índice de consulta **IX-1**: `CREATE INDEX ix_book_category_title_active ON library.book (category_id, title) WHERE deleted_at IS NULL;`.

### 2.3 Tabla `library.member`
Registro de socios lectores activos de la biblioteca.
- `id` (uuid, NOT NULL, PRIMARY KEY)
- `full_name` (varchar(200), NOT NULL)
- `document_number` (varchar(30), NOT NULL, UNIQUE)

**Restricciones e Índices:**
- Índice único **UX-2**: `CREATE UNIQUE INDEX ux_member_document_number ON library.member (document_number);`.

### 2.4 Tabla `library."user"`
Personal del sistema (Administradores y Bibliotecarios). El identificador `"user"` se declara entrecomillado por tratarse de una palabra reservada en PostgreSQL.
- `id` (uuid, NOT NULL, PRIMARY KEY)
- `username` (varchar(120), NOT NULL, UNIQUE)
- `password_hash` (varchar(512), NOT NULL)
- `role` (varchar(40), NOT NULL) — Valores permitidos: `admin`, `librarian`.

**Restricciones e Índices:**
- Índice único **UX-3**: `CREATE UNIQUE INDEX ux_user_username ON library."user" (username);`.

### 2.5 Tabla `library.loan`
Cabecera transaccional del préstamo de libros.
- `id` (uuid, NOT NULL, PRIMARY KEY)
- `member_id` (uuid, NOT NULL, FK hacia `library.member(id)`)
- `loaned_by_user_id` (uuid, NOT NULL, FK hacia `library."user"(id)`)
- `loaned_at` (timestamptz, NOT NULL) — Momento exacto del registro del préstamo.

**Restricciones e Índices:**
- Clave foránea **FK-2**: `fk_loan_member` hacia `member(id)` con `ON DELETE RESTRICT`.
- Clave foránea **FK-3**: `fk_loan_user` hacia `"user"(id)` con `ON DELETE RESTRICT`.
- Índice de consulta **IX-2**: `CREATE INDEX ix_loan_loaned_at ON library.loan (loaned_at);`.

### 2.6 Tabla `library.loan_item`
Detalle de libros prestados con valores congelados e inmutables (ADR-004, Artículo VIII).
- `id` (uuid, NOT NULL, PRIMARY KEY)
- `loan_id` (uuid, NOT NULL, FK hacia `library.loan(id)`)
- `book_id` (uuid, NOT NULL, FK hacia `library.book(id)`)
- `book_title` (varchar(250), NOT NULL) — Título congelado en el instante del préstamo.
- `isbn` (char(13), NOT NULL) — ISBN congelado.
- `category_name` (varchar(120), NOT NULL) — Nombre de categoría congelado (sin FK para preservar historia).
- `due_at` (timestamptz, NOT NULL) — Fecha límite de devolución (fijada a `loaned_at + 14 días`, RN-2).
- `returned_at` (timestamptz, NULL) — Momento en que se registra la devolución del ejemplar.

**Restricciones e Índices:**
- Clave foránea **FK-4**: `fk_loan_item_loan` hacia `loan(id)` con `ON DELETE CASCADE`.
- Clave foránea **FK-5**: `fk_loan_item_book` hacia `book(id)` con `ON DELETE RESTRICT`.
- Clave única de prevención de duplicados **UX-4**: `CREATE UNIQUE INDEX ux_loan_item_loan_book ON library.loan_item (loan_id, book_id);` (un libro solo puede figurar una vez por préstamo).

---

## 3. Semilla de Categorías Iniciales (D-01 a D-05)

La migración inicial ejecutada por Artisan inserta obligatoriamente las 5 categorías oficiales con sus identificadores UUID fijos:

| Código | UUID Fijo | Nombre de la Categoría |
|---|---|---|
| **D-01** | `e1b1a1a1-0001-4000-8000-000000000001` | `Novela` |
| **D-02** | `e1b1a1a1-0002-4000-8000-000000000002` | `Ciencia` |
| **D-03** | `e1b1a1a1-0003-4000-8000-000000000003` | `Tecnología` |
| **D-04** | `e1b1a1a1-0004-4000-8000-000000000004` | `Historia` |
| **D-05** | `e1b1a1a1-0005-4000-8000-000000000005` | `Infantil` |

---

## 4. Columnas Clave Verificadas

El modelo físico asegura la presencia estricta de las columnas requeridas:
1. `book.id`
2. `book.title`
3. `book.author`
4. `book.isbn`
5. `book.category_id`
6. `book.total_copies`
7. `book.available_copies`
8. `book.deleted_at`
9. `member.id`
10. `member.full_name`
11. `member.document_number`
12. `user.username`
13. `user.role`
14. `loan.loaned_at`
15. `loan_item.due_at`
16. `loan_item.returned_at`

