# Prueba Técnica: React Sénior/Lead - Dashboard de Agregación de Mercados (FTS)

## 1. Contexto del Desafío (El Escenario)

Somos "FinTech Synthetic Solutions" (FTS), una plataforma que provee datos de mercado de alta frecuencia. Nuestro producto principal es un *dashboard* de agregación que consume múltiples fuentes de datos (REST, GraphQL, WebSockets) y las presenta en una interfaz unificada, reactiva y de rendimiento crítico.

El *stack* tecnológico actual de nuestro *legacy* (basado en `useEffect` y Redux monolítico) está fallando en escalar, presentando cuellos de botella en el *render pipeline* y una gestión de estado asíncrono insostenible.

Tu misión es construir el prototipo de la v2 de este *dashboard*, demostrando maestría en la arquitectura de aplicaciones React modernas, gestión de estado de servidor de alta complejidad y optimización de rendimiento a nivel de milisegundos.

---

## 2. Definición de las Fuentes de Datos (Mock)

Deberás simular (o *mockear* con MSW) tres servicios de API distintos:

1.  **API REST (Legacy Market Data):**
    * `GET /api/v1/markets?page={number}&limit={number}`
    * Respuesta paginada (`{ data: [Market...], meta: { totalPages, currentPage } }`).
    * Devuelve una lista de miles de instrumentos financieros (ej. 10,000+ ítems).
    * `Market { id: string, name: string, baseCurrency: string, quoteCurrency: string }`

2.  **API GraphQL (Portfolio & User Data):**
    * `POST /api/graphql`
    * Un *endpoint* que expone el *schema* de portafolios de usuario.
    * Permite `Query` para `userPortfolio` (que incluye `watchlists`) y `Mutation` para `addToWatchlist` y `removeFromWatchlist`.

3.  **API WebSocket (Real-Time Price Ticker):**
    * `wss://api.example.com/stream`
    * Un *stream* que emite continuamente actualizaciones de precios en formato JSON.
    * `{ type: 'PRICE_UPDATE', payload: { marketId: string, price: number, timestamp: number } }`

---

## 3. Requisitos Funcionales y Técnicos (El Núcleo)

### RF-01: El "Market Grid" Virtualizado (Manejo de REST)

El componente principal será una tabla (o cuadrícula) que muestre todos los mercados disponibles desde la API REST.

* **Técnico (Data Fetching):** Debes usar **TanStack Query (React Query)** para gestionar el estado de esta API.
* **Técnico (Infinite Loading):** Dado el volumen de datos, debes implementar **`useInfiniteQuery`** para *lazy loading* de los datos a medida que el usuario hace *scroll* (desplazamiento infinito), en lugar de paginación tradicional.
* **Técnico (Rendimiento de Renderizado):** Es **mandatorio** implementar **virtualización de listas** (ej. `@tanstack/react-virtual` o `react-window`) para el *grid*. El DOM **no debe** contener nodos para los ítems que no están actualmente en el *viewport*.

### RF-02: Hidratación del Estado en Tiempo Real (Manejo de WebSockets)

El *grid* del RF-01 debe reflejar los precios en tiempo real.

* **Técnico (Conectividad):** Debes establecer y mantener una conexión con el *stream* WebSocket. Se espera que esta lógica esté encapsulada en un *custom hook* (ej. `useMarketStream`) que gestione el ciclo de vida de la conexión (conexión, desconexión, re-conexión).
* **Técnico (State Hydration):** A medida que se reciben mensajes del WebSocket (`PRICE_UPDATE`), **no debes** disparar un *refetch* de la API. En su lugar, debes actualizar *imperativamente* la caché de React Query (`queryClient.setQueryData`) para inyectar el nuevo precio en el *market* correspondiente.
* **Técnico (Memoización):** Debes demostrar una estrategia de memoización agresiva (`React.memo`, `useMemo`) en las filas del *grid* para asegurar que solo la fila cuyo precio cambió se vuelva a renderizar, evitando la re-renderización de las miles de otras filas en el *grid* virtualizado.

### RF-03: Gestión de Portafolio (Manejo de GraphQL y Mutaciones)

El usuario debe poder gestionar su *watchlist* personal, la cual se obtiene de la API de GraphQL.

* **Técnico (Data Fetching GQL):** Debes integrar un cliente de GraphQL (Apollo Client, URQL, o el mismo React Query) para consumir el *endpoint* de GraphQL y obtener el `userPortfolio`.
* **Técnico (Mutaciones):** El usuario debe poder hacer clic en un ícono "guardar" en el *grid* (RF-01) para añadir ese mercado a su *watchlist*. Esto debe disparar una `Mutation` de GraphQL (`addToWatchlist`).
* **Técnico (Optimistic UI):** La implementación de `addToWatchlist` **debe** ser una **actualización optimista**. El ícono debe cambiar a "guardado" instantáneamente, *antes* de que el servidor confirme la mutación. La UI debe manejar el *rollback* (reversión) en caso de que la mutación falle.
* **Técnico (Invalidación de Caché):** Tras una mutación exitosa (o un *refetch* tras un *rollback* fallido), la caché del `userPortfolio` (la *query* de GraphQL) debe ser invalidada para asegurar la consistencia de los datos.

### RF-04: Arquitectura, Pruebas y Rendimiento

* **Técnico (Code Splitting):** La vista de "Portafolio Detallado" (que consume GraphQL) debe ser un componente cargado de forma diferida (lazy loaded) usando `React.lazy()` y `Suspense` para dividir el *bundle* de la aplicación.
* **Técnico (Bundling):** El proyecto debe ser inicializado usando **Vite**. No se acepta Create React App (CRA).
* **Técnico (Testing - Mocking):** Es **mandatorio** el uso de **Mock Service Worker (MSW)** para interceptar y *mockear* todas las peticiones (REST y GraphQL) a nivel de red para las pruebas.
* **Técnico (Testing - Integration):** Debes escribir pruebas de integración (React Testing Library) que cubran el flujo completo del RF-03 (renderizar el *grid*, simular clic en "guardar", verificar la UI optimista y el *mock* de la mutación).
* **Técnico (Testing - Hooks):** Debes incluir pruebas unitarias (Vitest/Jest) para el *hook* `useMarketStream` (RF-02), *mockeando* la dependencia del WebSocket (`mock-socket` o similar).
* **Técnico (Error Handling):** Debes implementar `ErrorBoundary` (componentes de frontera de error) de React para gestionar fallos catastróficos en el *render* del *grid* o del portafolio.

---

## 4. Criterios de Evaluación

Se evaluará tu solución con base en los siguientes pilares:

1.  **Gestión de Estado Asíncrono (Server State):**
    * Maestría en el uso de TanStack Query (no solo `useQuery`, sino `useInfiniteQuery`, `queryClient`, `setQueryData`).
    * Calidad de la estrategia de "Optimistic UI" y la lógica de *rollback*.
    * Eficiencia en la invalidación y re-hidratación de la caché.

2.  **Rendimiento del Lado del Cliente (Client-Side Performance):**
    * Implementación correcta de la virtualización (demostrar que el DOM no crece).
    * Uso justificado y efectivo de la memoización (`React.memo`, `useMemo`).
    * Impacto de la hidratación del WebSocket en el *main thread* (debe ser mínimo).

3.  **Arquitectura de Software y Código Limpio:**
    * Uso estricto de **TypeScript** (ausencia de `any`, tipos inferidos, tipos genéricos para *hooks* y componentes).
    * Estructura del proyecto (colocación de estado, separación de la lógica de *fetching* de la UI, encapsulación en *hooks*).
    * Calidad de las pruebas (uso de MSW, cobertura de flujos críticos).

4.  **Conocimiento del Ecosistema:**
    * Configuración correcta de Vite.
    * Implementación de `React.lazy` y `Suspense`.
    * Manejo del ciclo de vida de los WebSockets.

## 5. Entregable

Debes entregar la URL de un repositorio de Git (GitHub) con el código fuente completo. El `README.md` debe incluir instrucciones detalladas para la instalación (`npm install`), ejecución del proyecto (`npm run dev`) y ejecución de la suite de pruebas (`npm run test`).