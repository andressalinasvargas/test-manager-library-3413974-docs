# ADR-005 · Arquitectura Hexagonal desacoplada en PHP con Laravel

- **Estado:** aceptada
- **Contexto:** El reto técnico exige implementar el backend en PHP con Laravel. La convención por defecto de Laravel (MVC con modelos Eloquent en `app/Models`, controladores directos y lógica de negocio acoplada al ORM) viola los principios de la Constitución (**Artículo I** y **Artículo II**), dificultando probar las invariantes de negocio sin una base de datos activa y generando fuerte dependencia del framework.
- **Decisión:** Implementar **Arquitectura Hexagonal (Puertos y Adaptadores)** dentro de la estructura estándar de Laravel:
  1. `app/Domain/`: Clases y objetos de valor en PHP puro (strict types), sin dependencias de `Illuminate` ni Eloquent.
  2. `app/Application/Ports/`: Interfaces abstractas para persistencia, reloj, seguridad y almacenamiento.
  3. `app/Application/Services/`: Casos de uso de negocio (`LoginService`, `BookService`, `LoanService`, etc.).
  4. `app/Adapters/`: Implementaciones concretas sobre PostgreSQL, JWT y disco local.
  5. `app/Providers/AppServiceProvider.php`: Punto único de enlace de dependencias en el contenedor de servicios de Laravel (**Artículo III**).
- **Alternativas descartadas:**
  - *MVC tradicional de Laravel con Eloquent:* Descartado porque acopla las entidades a la base de datos e impide cumplir los Artículos I, II y VI.
  - *Microframework Slim / Lumen:* Descartado para aprovechar la solidez de Laravel en migraciones de Artisan, enrutamiento, CLI y manejo de peticiones sin perder la pureza del dominio.
- **Consecuencias:**
  - El dominio es 100% testeable en milisegundos con PHPUnit puro.
  - La base de datos PostgreSQL y el esquema `library` pueden evolucionar sin modificar una sola regla de negocio.
  - El código cumple rigurosamente con los principios de desarrollo guiado por especificación (SDD).
- **Cómo se verifica:** Pruebas de arquitectura que aseguran que `app/Domain` no importa clases de Laravel ni de `app/Adapters`.

