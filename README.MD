# Desafío de Arquitectura de Frontend: "Proyecto Quasar" - El Framework de Composición de Finanzas

## 1. El Contexto del Negocio y El Escenario Técnico

Has sido seleccionado para liderar como Arquitecto de Frontend (Staff/Principal Engineer) el "Proyecto Quasar".

"Quasar" no es una aplicación; es una **plataforma de composición de UI en tiempo real** para instituciones financieras de primer nivel (Tier 1). Nuestra base de código única de React debe ser capaz de generar cientos de *aplicaciones de trading* distintas y de marca blanca (white-label) para nuestros clientes (bancos, fondos de cobertura).

La plataforma es un *dashboard* "infinito" donde los usuarios finales (traders) pueden componer su espacio de trabajo arrastrando y soltando *widgets* (módulos). Estos módulos son desarrollados y desplegados de forma independiente por más de 10 equipos de ingeniería (escuadrones) especializados (ej. Gráficos de Mercado, Análisis de Sentimiento, Ejecución de Órdenes, Noticias en Vivo, Cumplimiento Normativo).

### Los 5 Pilares del Desafío Arquitectónico:

1.  **Federación Masiva (MFE):** Cientos de módulos (Micro-Frontends o MFEs) deben coexistir y ser cargados dinámicamente en tiempo de ejecución.
2.  **Multitenancy y Composición:** Cada cliente (inquilino) tiene una configuración única de *features*, *branding* (temas) y permisos. La plataforma debe componerse a sí misma en el *runtime* basándose en el perfil del cliente.
3.  **Estado Colaborativo y Sincronización:** Los *dashboards* son colaborativos. Un analista en Londres y un *trader* en Nueva York pueden estar modificando el *mismo* *layout* de *dashboard* simultáneamente.
4.  **Rendimiento Extremo (Zero-Lag):** En el mundo del *trading*, milisegundos importan. La plataforma debe manejar miles de actualizaciones de datos por segundo (vía WebSockets) sin degradar el *main thread* de la UI.
5.  **Resiliencia y Modo Offline:** Las conexiones de los clientes son poco fiables. La aplicación debe ofrecer una experiencia robusta en modo *offline* o de baja conectividad (lectura de datos cacheados, cola de comandos).

## 2. El Entregable: Dos Fases (El Manifiesto y El Núcleo)

Tu tarea se divide en dos entregables críticos que deben presentarse a un comité de arquitectura (tus pares).

---

### FASE 1: El Manifiesto de Arquitectura (El Documento de Diseño)

(Formato: `ARCHITECTURE.md` en la raíz del repositorio)

Este documento es el entregable más importante. Debe justificar todas las decisiones arquitectónicas fundamentales que permitirán a 10 equipos trabajar en paralelo. Debe ser extenso, detallado y anticipar problemas de segundo y tercer orden.

#### Sección 1: Estrategia de Federación y Composición del "Shell"

1.  **Modelo de Federación:**
    * **Decisión:** Elige y justifica tu estrategia de federación de módulos (ej. Webpack 5 Module Federation, Vite + ES Modules nativos, iFrames controlados, o una solución de *build-time* como los *workspaces* de un monorepo).
    * **Justificación:** Debate los *trade-offs* de tu elección, enfocándote en:
        * **Aislamiento de Dependencias:** ¿Cómo manejarás las dependencias compartidas (ej. `react`, `react-dom`, `design-system`)? ¿Singleton, múltiples versiones, o un *peer-dependency* forzado? ¿Cómo prevendrás el "infierno de dependencias" de los MFEs?
        * **Aislamiento de Estilos:** ¿Cómo garantizarás que los estilos de MFE-A no "filtren" y rompan MFE-B? (ej. CSS-in-JS con `StyleSheetManager`, Shadow DOM, CSS Modules con *scoping* forzado).
        * **Contrato de API del Módulo:** ¿Cómo se "registra" un MFE en la aplicación "Host" (el *shell*)?

2.  **El "Servicio de Configuración y Orquestación" (El Director):**
    * **Diseño:** Describe la arquitectura de un servicio (ya sea un microservicio o un JSON estático) que el *shell* consumirá al arrancar. Este servicio debe definir:
        * El *layout* del *dashboard* del usuario.
        * La lista de MFEs a los que tiene acceso (basado en *feature flags* del cliente).
        * La URL de manifiesto (ej. `remoteEntry.js`) para cada MFE.
    * **Propagación:** ¿Cómo se propaga esta configuración a través del árbol de React? (¿Contexto, Redux, un *singleton* de Zustand?).

#### Sección 2: El Modelo de Estado (La Hidra)

Esta es la sección más crítica. Debes diseñar una arquitectura de estado de múltiples cabezas.

1.  **Estado del "Host" (Global):**
    * **Definición:** El estado que el *shell* posee (ej. Auth, perfil del usuario, tema (token de diseño), configuración de red).
    * **Propagación:** ¿Cómo se proporciona este estado de forma segura y performante a los MFEs (que pueden o no compartir el mismo `React.Context`)?

2.  **Estado Compartido (Comunicación Inter-MFE):**
    * **El Problema:** MFE "Watchlist" emite un `ASSET_SELECTED: 'AAPL'`. MFE "Gráficos" y MFE "Noticias" deben reaccionar a esto, sin que "Watchlist" sepa que existen.
    * **Diseño:** Propón un patrón de comunicación desacoplado. ¿Un Bus de Eventos (`CustomEvent`) en el objeto `window`? ¿Un *hook* (`useGlobalEvent`) que abstrae un *singleton* de `tiny-emitter`? ¿Un *slice* de estado compartido en un *store* global de Zustand/Redux? Justifica los *trade-offs* en términos de *type-safety* (seguridad de tipos) y rastreabilidad (debugging).

3.  **Estado Efímero (UI Local):**
    * ¿Cómo forzarás (o recomendarás) a los equipos de MFE que gestionen su propio estado interno (ej. `useState`, `useReducer`) para prevenir la contaminación del *store* global?

4.  **Estado del Servidor (Data Fetching):**
    * **Estrategia:** Asumiendo el uso de **TanStack Query (React Query)**, ¿cómo gestionarás la *query cache*?
    * **Desafío:** Si MFE-A y MFE-B hacen `useQuery(['user-portfolio'])`, ¿compartirán la misma entrada de caché? ¿Cómo manejarás la invalidación de caché iniciada desde *otro* MFE? Propón un patrón para una `QueryClient` global inyectada.

5.  **Estado Colaborativo (Tiempo Real - CRDTs):**
    * **El Desafío:** El *layout* del *dashboard* (la posición y tamaño de los *widgets*) es un estado colaborativo.
    * **Diseño:** Propón una arquitectura para manejar la sincronización de estado concurrente. No es necesario implementar CRDTs, pero debes describir el flujo:
        * ¿Usarías un WebSocket "tonto" (LWW - Last-Write-Wins) o una biblioteca de CRDT (ej. `Yjs`, `Automerge`)?
        * Dibuja un diagrama de secuencia:
            1. Usuario A arrastra el Widget 1.
            2. El MFE del *dashboard* emite un cambio de "intención".
            3. ¿Qué servicio (el *shell*, un *hook* global) recibe esto?
            4. ¿Cómo se serializa y envía al "Servicio de Sincronización"?
            5. ¿Cómo se recibe, fusiona (merge) y re-propaga el estado a Usuario B sin causar un bucle de renderizado infinito?

#### Sección 3: Arquitectura de Rendimiento y Resiliencia

1.  **Renderizado y Arranque en Frío:**
    * **Decisión:** ¿SPA (Single Page Application) puro, SSR (Server-Side Rendering) del *shell*, o RSC (React Server Components)?
    * **Justificación:** Argumenta tu elección en el contexto de un *dashboard* dinámico y personalizado. ¿Cómo impacta esto en el TTI (Time to Interactive)? ¿Cómo harás *code-splitting* (división de código) no solo a nivel de ruta, sino a nivel de *widget*?

2.  **El "Kernel Offline" (Offline-First):**
    * **Diseño:** Propón una estrategia robusta para la resiliencia.
    * **Service Worker:** ¿Qué estrategia de caché usarás (`StaleWhileRevalidate` para los MFEs, `CacheFirst` para el *shell* y las librerías)?
    * **Persistencia de Datos:** ¿Qué datos persistirás en **IndexedDB**? (ej. la última configuración conocida del *layout*, datos de mercado cacheados).
    * **Cola de Mutaciones:** Si el usuario está *offline* e intenta ejecutar una orden (mutación), ¿cómo diseñarás una "cola de salida" (Outbox) que sincronice las mutaciones cuando la red vuelva?

3.  **Estrategia de Pruebas y CI/CD:**
    * **Pruebas de Contrato:** ¿Cómo asegurarás que MFE-A no rompa el contrato de API del que depende MFE-B? (ej. Pruebas de Contrato con PACT).
    * **Entorno de CI:** ¿Cómo probarás la *integración* completa en un *pipeline* de CI? (ej. Desplegar todos los MFEs a un entorno de *staging* y correr pruebas E2E (Cypress/Playwright) contra el *shell* orquestado).
    * **DevX (Developer Experience):** ¿Cómo podrá un desarrollador del equipo "Noticias" correr su MFE localmente, pero consumiendo los MFEs "Gráficos" y "Watchlist" de *producción* (o *staging*) para no tener que correr 10 proyectos localmente?

---

### FASE 2: El Núcleo del Prototipo (Prueba de Concepto - PoC)

(Formato: El repositorio de código)

No debes construir todos los *features* del manifiesto. Debes construir el **esqueleto funcional** que demuestre que los **puntos de riesgo** de tu arquitectura son viables.

Usa un **Monorepo** (ej. Turborepo/Nx) para estructurar este PoC.

#### PoC-1: El "Host Shell" y el Orquestador

* Una aplicación "Host" (ej. en `apps/shell`).
* Debe cargar y renderizar al menos dos MFEs remotos (ej. `apps/mfe-watchlist`, `apps/mfe-charts`) usando tu estrategia de federación (ej. Module Federation).
* **Requisito Clave:** La *composición* debe ser dinámica. El *shell* debe leer un `layout.json` (mock) al inicio y renderizar los MFEs y su posición basándose en ese JSON.

#### PoC-2: La Comunicación Inter-MFE Desacoplada

* Implementa el "Bus de Eventos" o patrón de estado compartido que diseñaste en el Manifiesto.
* **Demostración:** Al hacer clic en un ítem en `mfe-watchlist`, `mfe-charts` debe recibir el `assetId` y mostrarlo. `mfe-watchlist` **no debe** tener una importación directa o dependencia de `mfe-charts`.

#### PoC-3: El Stub del "Kernel Offline"

* No necesitas una implementación completa de IndexedDB.
* **Demostración:**
    1.  Implementa un **Service Worker** básico.
    2.  Haz que el *shell* haga un *fetch* de datos (ej. `GET /api/markets`).
    3.  El Service Worker debe interceptar esta llamada y cachear la respuesta (Estrategia: `StaleWhileRevalidate`).
    4.  Demuestra (en la consola de Chrome, simulando *offline*) que si la API falla o está *offline*, la aplicación sigue cargando los datos desde la caché del SW.

#### PoC-4: La Demostración de Aislamiento (Estilos y Enfoque)

* **Aislamiento de Estilos:** Demuestra que los estilos de `mfe-watchlist` (ej. `h1 { color: red; }`) no afectan a `mfe-charts` (ej. usando Shadow DOM o la técnica que hayas elegido).
* **Gestión de Foco (A11y):** Demuestra que la navegación por teclado (`Tab`) puede moverse *seamlessly* (sin interrupciones) desde el último elemento enfocable de `mfe-watchlist` al primer elemento enfocable de `mfe-charts` y viceversa, como si fueran una sola aplicación.

#### PoC-5: El "Stub" de Sincronización de Estado (El Desafío)

* No necesitas CRDTs.
* **Demostración:**
    1.  Abre la aplicación en **dos pestañas** del navegador.
    2.  Usa un WebSocket (puedes usar un servidor de ws simple o un *mock* usando `mock-socket`) o la `BroadcastChannel` API para simular la conexión.
    3.  Al cambiar el estado en la Pestaña 1 (ej. un *toggle* en el `mfe-watchlist`), ese cambio debe reflejarse **automáticamente** en la Pestaña 2 sin necesidad de refrescar. Esto prueba la viabilidad de la arquitectura de estado reactivo y colaborativo.

## 3. Criterios de Evaluación

* **Profundidad del Manifiesto (60%):** Tu capacidad para identificar problemas de segundo orden, justificar *trade-offs* complejos (no hay "respuestas correctas") y diseñar para la escalabilidad *organizacional* (DevX, CI/CD, equipos).
* **Viabilidad del PoC (40%):** ¿El código del PoC implementa exitosamente los patrones de alto riesgo descritos en el manifiesto?
* **Calidad del Código:** Se espera un dominio absoluto de TypeScript (Genéricos, Inferencia, Tipos Condicionales) y patrones de React (Hooks avanzados, Composición, Contexto) en el PoC.
* **Claridad:** ¿Son el manifiesto y el código claros, mantenibles y fáciles de entender para los 10 equipos que se unirán después de ti?

Moraleja: Hay que leer primero. Todo esto es caspa, ni yo soy capaz de hacerlo, se puede ir pa su casa ya