# ADR-006 · Frontend en React como Single Page Application con Vite y Nginx

- **Estado:** aceptada
- **Contexto:** La especificación exige desarrollar el frontend de Manager Library en **React**, consumiendo el contrato RESTful normalizado de la API de Laravel (`/api/v1`). Se requiere un empaquetado rápido, tiempos de recarga mínimos y despliegue desacoplado en contenedores Docker independientes.
- **Decisión:** Implementar una **Single Page Application (SPA) con React**, empaquetada mediante **Vite** y desplegada en producción a través de un servidor ligero **Nginx** en el puerto `5173`:
  1. Uso de Vite como herramienta de construcción por su velocidad y soporte nativo de módulos ES.
  2. Arquitectura de componentes modular en React (`App.jsx`) con estado reactivo para catálogos, socios, préstamos, devoluciones, préstamos vencidos y reportes.
  3. Comunicación exclusiva mediante peticiones HTTP `fetch` hacia `/api/v1` con cabecera `Authorization: Bearer <token>`.
  4. Nginx configurado con enrutamiento SPA (`try_files $uri $uri/ /index.html;`) para garantizar navegación fluida sin recargas completas.
- **Alternativas descartadas:**
  - *Next.js / SSR:* Descartado para evitar sobrecarga de servidor de Node.js en tiempo de ejecución, manteniendo la infraestructura simple y estática.
  - *Create React App (CRA):* Descartado por ser una herramienta obsoleta con dependencias pesadas y tiempos lentos de compilación.
- **Consecuencias:**
  - Desacoplamiento total entre la interfaz de usuario en React y el backend en Laravel.
  - Empaquetado en Docker de alta velocidad con imágenes pequeñas (Alpine Nginx).
  - Experiencia de usuario ágil y responsiva para bibliotecarios y administradores durante la feria.
- **Cómo se verifica:** El contenedor `manager-library-app` sirve la interfaz en `http://localhost:5173` y responde correctamente sin errores de consola al consumir la API de Laravel.
