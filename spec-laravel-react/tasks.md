# Tareas — Manager Library (PHP Laravel + React)

Matriz detallada de tareas del proyecto **Manager Library** adaptada al stack **PHP (Laravel) + React (Vite)**, estructurada bajo el principio de verificación tangible con evidencia ejecutable (**Artículo XII**).

---

## 1. Matriz de Tareas (T-01 a T-23)

| # | Tarea | Componente | Dependencia | Evidencia de Cumplimiento en Laravel / React |
|---|---|---|---|---|
| **T-01** | Esqueleto del backend en Laravel y endpoint de salud | Backend (`api`) | — | `GET /api/v1/health` responde `{"status":"ok"}`; el contenedor `manager-library-api` arranca limpiamente en puerto 8000. |
| **T-02** | Migración inicial de base de datos en Laravel | Persistencia / DDL | T-01 | `php artisan migrate` crea las 6 tablas en esquema `library`, FK-1 a FK-6, índices únicos y siembra las 5 categorías oficiales (D-01 a D-05). Sin tablas de framework ni created_at/updated_at (§8). |
| **T-03** | Objetos de valor del dominio en PHP (`Isbn`, `Quantity`, `DateRange`) | Dominio | T-01 | Pruebas unitarias en PHPUnit validando ISBN de 13 dígitos, cantidad entera >= 0 y rangos cronológicos válidos sin dependencias de framework (Artículo I). |
| **T-04** | Agregado de dominio `Book` | Dominio | T-03 | Pruebas unitarias de `withdraw()` y `restock()`. Regla RN-1 (`0 <= available_copies <= total_copies`) y baja lógica `markDeleted()`. |
| **T-05** | Agregado de dominio `Loan` y `LoanItem` | Dominio | T-04 | Pruebas con `Clock` simulado verificando límite de 1 a 3 libros (RN-3), 14 días calendario (RN-2) y cálculo dinámico de estados `open`, `overdue`, `closed` (Artículo VII). |
| **T-06** | Entidades de dominio `Member` y `User` | Dominio | T-03 | Normalización de usuarios a minúsculas, validación de roles (`admin`, `librarian`), unicidad de documento de identidad y validación de elegibilidad de préstamo. |
| **T-07** | Puertos y servicios de autenticación (`LoginService`, `CreateLibrarianService`) | Aplicación | T-06 | Pruebas contra fakes para HU-01 y HU-02: emisión de token JWT de 8 horas (RN-5) y rechazo de contraseñas inválidas con `INVALID_CREDENTIALS` (CA-01.2). |
| **T-08** | Puertos y servicios de libros (`BookService`) | Aplicación | T-04 | Pruebas para HU-03 y HU-04: alta, edición, búsqueda y baja lógica restringida a administradores (CA-04.1). |
| **T-09** | Servicio de préstamo de libros (`LoanService`) | Aplicación | T-05, T-08 | Verificación completa de HU-06: descuento atómico de existencias, rechazo si no hay copias (`BOOK_OUT_OF_STOCK`), si excede cupo (`MEMBER_LIMIT_EXCEEDED`) o si tiene moras (`MEMBER_HAS_OVERDUE_LOANS`). |
| **T-10** | Servicios de devolución y préstamos vencidos | Aplicación | T-09 | Verificación de HU-07 y HU-09: devolución unitaria de línea, incremento de stock disponible, idempotencia ante doble devolución (`ITEM_ALREADY_RETURNED`, CA-07.2) y listado de moras activas. |
| **T-11** | Servicio de reporte analítico de libros más prestados | Aplicación | T-09 | Verificación de HU-10: cálculo del Top 10 sobre valores históricos congelados de `loan_item` (ADR-004), preservando título original aunque el libro haya sido editado. |
| **T-12** | Adaptadores de persistencia PostgreSQL en Laravel | Infraestructura | T-02, T-08 | `PostgresBookRepository`, `PostgresMemberRepository` y `PostgresUserRepository` operando con sentencias SQL nativas de PDO sobre el esquema `library`. |
| **T-13** | Repositorio de préstamos con control de concurrencia | Infraestructura | T-12, T-09 | `PostgresLoanRepository` y `DbTransactionRunner` con bloqueo transaccional (`FOR UPDATE`) para evitar sobre-préstamos concurrentes ante dos peticiones simultáneas del último ejemplar (RNF-2). |
| **T-14** | Adaptador de seguridad: hash y emisor de tokens JWT | Seguridad | T-07 | Emisión de tokens Bearer JWT firmados con clave simétrica HS256 (`JWT_SECRET`) y verificación de expiración de 8 horas. Contraseñas seguras nunca expuestas (RNF-3). |
| **T-15** | Adaptador de portadas en disco | Infraestructura | T-08 | Almacenamiento de archivos en `storage/app/covers/`. Acepta PNG/JPG hasta 2 MB; rechaza archivos superiores con código 422. Si no se envía portada, `cover_key` queda nulo. |
| **T-16** | Controlador HTTP RESTful y enrutamiento en Laravel | HTTP / API | T-10, T-11, T-14, T-15 | Enrutamiento de los 21 endpoints bajo `/api/v1` en `routes/api.php`, middleware `AuthenticateWithJwt` y formato uniforme de error con los 14 códigos oficiales. |
| **T-17** | Composición IoC y arranque inicial en Laravel | Bootstrap | T-16 | Enlace de puertos y adaptadores en `AppServiceProvider`. Comando `library:bootstrap-admin` para crear el usuario administrador inicial desde variables de entorno. |
| **T-18** | Endurecimiento de restricciones en motor PostgreSQL | Base de Datos | T-12 | Verificación de `ck_book_available_range`, índices únicos parciales e integridad referencial garantizada por motor ante cualquier inserción externa (Artículo VI). |
| **T-19** | Pruebas de arquitectura hexagonal en Laravel | Calidad | T-17 | Garantía estricta de que `Domain` no importa componentes de framework y que `Application` no depende de `Adapters` (Artículos I y II). |
| **T-20** | Orquestación completa con Docker Compose (`infra`) | Infraestructura | T-17 | `docker compose up -d` levanta los contenedores `db` (Postgres 16), `api` (Laravel) y `app` (React SPA) sobre la red privada `library-net`. Base de datos vacía al inicio (ADR-001). |
| **T-21** | Sembrador de datos de demostración (`tool`) | Herramientas | T-17 | Script `seed.php` en PHP CLI que consume exclusivamente la API REST para sembrar 1 bibliotecario, 40 libros, 10 socios, 15 préstamos y 3 vencidos sin llamadas SQL directas. |
| **T-22** | Aplicación Frontend en React SPA (`app`) | Frontend | T-16 | Interfaz completa en React con Vite y Nginx en puerto 5173: Login, Catálogo con filtros y búsqueda, Socios, Préstamos con validación de stock, Devoluciones, Vencidos y Reportes. |
| **T-23** | Sitio público estático institucional (`page`) | Presentación | — | Página web estática `index.html` moderna, responsiva, sin llamadas directas a la API y sin exponer credenciales ni tokens. |

---

## 2. Definición de Hecho (DoD) de la Solución

El proyecto alcanza el nivel de excelencia técnica al completar de punta a punta las tareas desde **T-01 hasta T-23**, garantizando que el sistema esté listo para su demostración en vivo durante la feria técnica.

