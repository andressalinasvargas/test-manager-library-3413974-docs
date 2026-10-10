# Constitución — Manager Library (PHP Laravel + React)

Adaptación oficial del documento fundamental de principios innegociables para el ecosistema PHP con Laravel en el backend y React con Vite en el frontend, en el marco de la metodología Spec-Driven Development (SDD).

Un principio sin mecanismo de comprobación es solo una buena intención. Cada artículo establece su justificación arquitectónica, su impacto directo en el diseño y su método exacto de verificación en Laravel y React.

---

## Artículo I — El dominio no depende de nada en PHP

El paquete `app/Domain/` de Laravel está completamente aislado. No importa ningún componente del framework Laravel (Illuminate), ni Eloquent ORM, ni controladores HTTP, ni paquetes de terceros de Composer. Solo hace uso de las capacidades estándar de PHP 8.2+ con tipos estrictos (`declare(strict_types=1);`), clases inmutables (readonly classes), enumeraciones nativas y tipos escalares.

- **Por qué:** Es la única garantía de que las reglas e invariantes del negocio (como las restricciones en `Isbn`, `Quantity`, `DateRange`, `Book` y `Loan`) puedan probarse de forma pura mediante pruebas unitarias en PHPUnit sin requerir contenedores de dependencias de Laravel, sin levantar PostgreSQL y sin depender de Artisan.
- **Verificación:** Un análisis estático con PHPStan / Pest Architecture Tests que comprueba que ningún archivo dentro del namespace `App\Domain` importe clases del namespace `Illuminate\*` o de `App\Adapters`.
- **Trazabilidad:** Relacionado con T-03, T-04, T-05, T-06, RN-1, RN-2 y RN-3.

---

## Artículo II — El núcleo declara sus puertos; el mundo los implementa en Laravel

La capa de aplicación en PHP (`app/Application/`) solo conoce las entidades y objetos de valor del dominio. Todas las necesidades externas hacia la infraestructura (repositorios de persistencia, reloj del sistema, algoritmos de hash criptográfico, emisión de tokens JWT, almacenamiento de portadas en disco o S3) se declaran como interfaces estrictas denominadas **Puertos** dentro de `app/Application/Ports/`.

Ninguna clase de Laravel (como `Request`, `Response`, `Model` de Eloquent o `DB`) puede aparecer en la firma, parámetros o tipos de retorno de un puerto en PHP.

- **Por qué:** Permite que los casos de uso (como `LoginService`, `CreateLibrarianService`, `BookService`, `MemberService`, `LoanService` y `ReportService`) sean completamente independientes de los controladores de Laravel y del motor de base de datos.
- **Verificación:** Pruebas unitarias de aplicación que ejecutan los casos de uso contra implementaciones falsas (Fakes/Mocks en memoria) sin tocar la base de datos PostgreSQL y sin invocar comandos de Artisan.
- **Trazabilidad:** Verificado en T-07, T-08, T-09, T-10, T-11 y respaldado por ADR-001 y ADR-002.

---

## Artículo III — Un único punto de composición en Laravel

En Laravel, el contenedor de servicios de inversión de control (IoC) gestionado por `AppServiceProvider` (`app/Providers/AppServiceProvider.php`) es el único punto autorizado donde un Puerto de aplicación se enlaza con su Adaptador de infraestructura concreto mediante `$this->app->bind()`.

Ningún controlador, comando de Artisan o servicio puede instanciar directamente un adaptador de persistencia como `PostgresBookRepository` o `PostgresLoanRepository` usando `new`.

- **Por qué:** Asegura un acoplamiento débil y respeta el principio de inversión de dependencias (DIP) facilitando sustituir implementaciones sin alterar la lógica de negocio.
- **Verificación:** Inspección estática que valida que las referencias a `App\Adapters\*` solo existan en `AppServiceProvider` o pruebas de integración de Laravel.
- **Trazabilidad:** T-17, T-19 y principios fundamentales de arquitectura hexagonal.

---

## Artículo IV — Principios SOLID en PHP y Laravel

| Principio | Aplicación en Laravel y PHP | Método de Verificación |
|---|---|---|
| **SRP (Responsabilidad Única)** | Cada caso de uso en `app/Application/Services/` implementa una única acción del sistema. `LoanService` presta y devuelve ejemplares y no maneja usuarios ni reportes. | Una clase con más de una razón de cambio se refactoriza en servicios separados. |
| **OCP (Abierto / Cerrado)** | Agregar un nuevo almacenamiento de portadas (de disco local a Cloud Storage) no altera `BookService` ni `Book`. | Se agrega una nueva clase que implementa `CoverStorage` en `Adapters` y se reconfigura en `AppServiceProvider`. |
| **LSP (Sustitución de Liskov)** | Las implementaciones en memoria (Fakes) de los repositorios respetan los mismos contratos que los adaptadores PostgreSQL. | La misma suite de pruebas pasa tanto con fakes como con `PostgresBookRepository`. |
| **ISP (Segregación de Interfaces)** | Puertos pequeños y específicos: `Clock`, `PasswordHasher`, `TokenIssuer`, `CoverStorage`, `LoanReader`, `ReportReader`. | Ningún adaptador implementa métodos vacíos o innecesarios. |
| **DIP (Inversión de Dependencias)** | Los casos de uso dependen exclusivamente de las abstracciones definidas en `app/Application/Ports/`. | Los puertos residen en la capa de aplicación y nunca en infraestructura. |

---

## Artículo V — El esquema es propiedad de las migraciones de Laravel

El contenedor de infraestructura `manager-library-db` arranca con un motor PostgreSQL 16 completamente **vacío**. Ningún archivo SQL ni sentencia `CREATE TABLE` existe en el repositorio de infraestructura (`infra`).

Todo el lenguaje de definición de datos (DDL) —incluyendo el esquema `library`, las seis tablas en singular (`category`, `book`, `member`, `user`, `loan`, `loan_item`), claves foráneas, restricciones `CHECK`, índices únicos parciales y la semilla de las 5 categorías— es propiedad exclusiva de las migraciones ejecutadas por Artisan en Laravel (`php artisan migrate`).

- **Por qué:** Garantiza la reproducibilidad y evolución controlada del modelo de datos en cualquier entorno.
- **Verificación:** El repositorio `infra` no contiene ningún archivo DDL, cumpliendo estrictamente con ADR-001 y T-02.

---

## Artículo VI — Las invariantes que la base de datos puede imponer, se imponen en PostgreSQL

Una regla de integridad que solo existe en el código de PHP es una defensa frágil. Toda invariante que pueda ser verificada por el motor relacional de PostgreSQL se declara directamente en la migración de Laravel:
- Restricción de chequeo: `CHECK (available_copies >= 0 AND available_copies <= total_copies)`.
- Restricción de unicidad parcial: `UNIQUE (isbn) WHERE deleted_at IS NULL` para libros no eliminados.
- Restricción de unicidad en miembro: `UNIQUE (document_number)`.
- Restricción de unicidad en usuario: `UNIQUE (username)`.
- Restricción de unicidad en préstamo: `UNIQUE (loan_id, book_id)`.

- **Por qué:** Protege la consistencia de los datos frente a accesos concurrentes e inserciones directas, cumpliendo con ADR-002, ADR-003 y RNF-2.
- **Verificación:** Intentos de inserción inválidos fallan a nivel de motor arrojando excepciones capturadas por Laravel.

---

## Artículo VII — Lo derivado se calcula, no se almacena

El estado de un préstamo (`open`, `overdue`, `closed`) y los días de retraso se calculan en tiempo de ejecución en PHP comparando la fecha de vencimiento (`due_at`) con la hora provista por el puerto `Clock`. En la base de datos no existe ninguna columna `status` ni `days_late` en las tablas `loan` o `loan_item`.

- **Por qué:** Elimina inconsistencias y redundancia de datos. El paso del tiempo no debe requerir cronjobs de actualización masiva de estados.
- **Verificación:** Inspección de las columnas físicas en `data-model.md` §3 y la migración de Laravel.

---

## Artículo VIII — Los valores congelados no siguen al catálogo

La línea de préstamo (`loan_item`) almacena como instantánea histórica el título del libro (`book_title`), el ISBN (`isbn`) y el nombre de la categoría (`category_name`) en el instante exacto en que se realiza el préstamo.

Si un libro es editado posteriormente en el catálogo vivo, los reportes analíticos de préstamos y el historial de transacciones continúan reflejando los valores inmutables capturados al prestar.

- **Por qué:** Mantiene la fidelidad histórica y la auditoría inmutable del acervo circulante según ADR-004 y HU-10.
- **Verificación:** Prueba de integración que modifica el título de un libro tras ser prestado y confirma que el reporte analítico conserva el título original.

---

## Artículo IX — Ningún secreto entra al repositorio ni a un log

Ninguna contraseña en texto claro, clave secreta de JWT (`JWT_SECRET`) o credencial de acceso a PostgreSQL se almacena en el control de versiones. Todas las credenciales sensibles son suministradas mediante variables de entorno a través del archivo `.env`.

En los repositorios de Laravel (`api`) e infraestructura (`infra`), el archivo `.env` está expresamente excluido mediante `.gitignore`, versionando únicamente `.env.example`.

- **Por qué:** Cumplimiento de la regla innegociable de seguridad y prevención de fugas de datos sensibles (RNF-3).
- **Verificación:** Ningún log de Laravel ni respuesta JSON de la API expone contraseñas, hashes de contraseñas ni claves privadas.

---

## Artículo X — Gana el motor

Si existe alguna discrepancia entre la documentación textual del modelo de datos y las restricciones efectivas aplicadas en la base de datos PostgreSQL, la base de datos prevalece y la documentación debe ajustarse inmediatamente.

---

## Artículo XI — Idioma

Todo el código fuente en PHP y React (nombres de clases, interfaces, métodos, variables, identificadores, rutas de la API y nombres de columnas de base de datos) se escribe estrictamente en **inglés**.

La documentación técnica, los mensajes de error retornados por la API hacia el usuario final y los textos de la interfaz gráfica en React se redactan en **español**.

---

## Artículo XII — Nada se da por hecho sin evidencia

Una tarea del plan de trabajo (`tasks.md`, de T-01 a T-23) se considera completada únicamente cuando su evidencia tangible existe, se puede inspeccionar y se ejecuta exitosamente. No se admiten tareas marcadas como hechas por simple promesa de código.

