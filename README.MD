# Prueba de React Nivel Arquitecto/Divinidad: El Motor de Visualización "Kairós"

## 1. El Escenario: El Fracaso del DOM

Estás aplicando para ser el Arquitecto de Frontend Principal en "Kairós", una firma de *trading* algorítmico de ultra-alta frecuencia (U-HFT).

Nuestro problema es simple: **el DOM es nuestro cuello de botella y `setState` es nuestro enemigo**.

Nuestros *traders* necesitan visualizar el **Libro de Órdenes (Order Book) L3** (datos de mercado completos) para el par `ETH/BTC`. Este *stream* de datos, que recibimos vía WebSocket en formato binario (FlatBuffers/Protobuf), emite **decenas de miles de deltas (cambios) por segundo**.

Hemos intentado con 3 equipos sénior construir esto en React. Todos han fallado.
* **Intento 1 (Fracaso):** Un `map()` sobre un array en el estado de React. El navegador se congela en 200ms.
* **Intento 2 (Fracaso):** Virtualización de DOM (`react-window`). El rendimiento es mejor, pero el *diffing* del DOM virtual sigue siendo demasiado lento y el *overhead* de 10,000 componentes de React en memoria destruye el *garbage collector*.
* **Intento 3 (Fracaso):** Renderizado en `<canvas>` usando una biblioteca (ej. `react-konva`). El *bridge* de React al estado del canvas se satura. El *data fetching* en el *main thread* bloquea la UI.

Tu misión es diseñar y construir el **núcleo del motor de renderizado** de este componente. Debes tratar a React no como un *framework*, sino como un orquestador de *scheduling* (planificación) y renderizado.

**Requisito No Negociable:** El componente final debe renderizar 60fps estables, sin *jank* (saltos), mientras procesa 20,000 actualizaciones de datos por segundo, y debe permitir la interacción del usuario (como hacer clic en un precio) con una latencia de < 50ms.

---

## 2. El Desafío: El Prototipo del "Núcleo del Reactor"

Tu tarea es construir un **único componente de React (`<L3OrderBook />`)** que cumpla con una arquitectura de rendimiento extremo.

### Arquitectura Mandatoria (Los Pilares)

Tu solución **debe** implementar los siguientes tres patrones arquitectónicos.

#### Pilar 1: El Pipeline de Datos (Fuera del Main Thread)

El *main thread* (hilo principal) de JavaScript es solo para la UI. El procesamiento de datos debe ser completamente asíncrono y paralelo.

1.  **Web Worker de Red:** El *stream* del WebSocket **debe** ser manejado en un **Web Worker** dedicado. Este *worker* es el único responsable de la conexión de red.
2.  **Módulo WebAssembly (WASM):** Los datos llegan en formato binario (simula un `ArrayBuffer`). JavaScript es demasiado lento para decodificar y calcular los deltas. **Debes** escribir (o simular la existencia de) un módulo **Rust/C++/Go compilado a WASM** que se cargue dentro del Web Worker.
3.  **Flujo en el Worker:**
    * WebSocket recibe `ArrayBuffer`.
    * El Worker pasa el *buffer* (idealmente *zero-copy*) al módulo WASM.
    * El módulo WASM decodifica el *buffer*, calcula los deltas contra el estado anterior del libro de órdenes (que también mantiene en la memoria de WASM), y devuelve una **estructura de datos de UI mínima** (ej. solo los 100 precios superiores/inferiores y sus volúmenes agregados).
    * El Worker envía este *snapshot* de UI al *main thread*.

#### Pilar 2: El Modelo de Estado (Anti-`useState`)

El estado de alta frecuencia **no puede** vivir en el *store* de React.

1.  **Suscripción de Estado Externo:** El estado del libro de órdenes (el *snapshot* enviado desde el Worker) **debe** ser gestionado fuera de React (ej. en una variable `let store` en el módulo).
2.  **`useSyncExternalStore`:** Debes usar el *hook* **`useSyncExternalStore`** para suscribir tu componente de React a este *store* externo. Esta es la única forma de puentear el estado externo con el *scheduler* de React sin causar *tearing* (desgarro de UI).
3.  **Prohibición:** Queda **estrictamente prohibido** el uso de `useState`, `useReducer`, Redux o Zustand para el estado del libro de órdenes. Estos introducen *overhead* y *batching* (agrupación) que no controlamos.

#### Pilar 3: El Motor de Renderizado (DOM Prohibido)

Renderizar 2,000 nodos de DOM (1,000 *bids* + 1,000 *asks*) 60 veces por segundo es imposible.

1.  **Renderizado Imperativo en `<canvas>` o WebGL:** El *output* visual del componente **debe** ser un único elemento `<canvas>`.
2.  **El "Reconciliador" Personalizado:** Aquí está el núcleo del desafío. **No puedes** usar un `useEffect` para pintar en el canvas.
    * **Opción A (Extrema):** Usar el paquete `react-reconciler` para escribir tu propio **motor de renderizado de React para Canvas**. Tu componente no devolvería `<div>` o `<span>`, sino `<OrderBookRow>` o `<DepthChartBar>`, y tu reconciliador sabría cómo traducir estas "instancias de React" en comandos de dibujo imperativos (`ctx.fillRect`, `ctx.fillText`).
    * **Opción B (Aceptable):** Usar `react-konva` o `pixi.js` como tu "motor de renderizado", **pero** debes justificar en un documento (`ARCHITECTURE.md`) por qué su modelo de *batching* de estado es superior a un `useEffect` e implementar el Pilar 2 (`useSyncExternalStore`) para alimentarlo.
3.  **Sincronización:** El renderizado en el canvas **debe** estar sincronizado con las actualizaciones de estado (Pilar 2) y el *frame* del navegador (usando `requestAnimationFrame` si no usas la Opción A).

### Fase de Implementación (PoC)

Debes entregar un repositorio con:

1.  **`ARCHITECTURE.md`:** Un documento que justifique cada elección técnica (por qué WASM, por qué `useSyncExternalStore`, por qué `react-reconciler` o por qué no).
2.  **El Módulo WASM (Mock):** No necesitas ser un experto en Rust. Puedes proveer un *stub* (simulacro) de `add.js` que *simule* ser el módulo WASM compilado. Lo importante es la arquitectura de *carga* (`await import('module.wasm')`) dentro del Worker.
3.  **El Web Worker (`data.worker.js`):** El *script* que maneja el *mock* del WebSocket y el *mock* del WASM.
4.  **El Componente (`L3OrderBook.jsx`):**
    * Implementa `useSyncExternalStore` para suscribirse al *store* que el Worker actualiza.
    * Implementa el motor de renderizado en `<canvas>` (Pilar 3).
    * Debe mostrar visualmente el libro de órdenes (una lista de precios y volúmenes) y una barra de "profundidad" (un gráfico de barras simple).
5.  **La Demostración de Concurrencia:**
    * El componente **debe** tener un `<input type="text">` *sobre* el canvas (como un DOM *overlay*).
    * **Prueba de Fuego:** Mientras el libro de órdenes se actualiza a 60fps (simulado), el usuario **debe** poder escribir en el `<input>` **sin ningún lag** o *jank*. Esto probará que tus actualizaciones de estado de baja prioridad (el libro) no están bloqueando la entrada de alta prioridad del usuario (el *input*).
6.  **El Perfil de Rendimiento (Deliverable):**
    * Debes incluir capturas de pantalla o un video de la pestaña **Performance** de Chrome DevTools, demostrando que:
        * La aplicación mantiene 60fps.
        * El *Main Thread* no está bloqueado (no hay "bloques rojos" largos).
        * El trabajo pesado (Scripting) ocurre en el *Worker Thread*.
        * La memoria (JS heap) es estable y no hay *memory leaks*.

---

## 3. Criterios de Evaluación

* **¿Fallaste? (Sí/No):** ¿La UI se congela, cae por debajo de 60fps, o tiene *lag* en el *input*? Si es así, es un fallo. No hay puntos intermedios.
* **Dominio de la Arquitectura de React:** ¿Entendiste *por qué* `useSyncExternalStore` es la herramienta correcta? ¿Entendiste *por qué* el estado debe vivir fuera de React?
* **Dominio de la Concurrencia:** ¿Demostraste una comprensión profunda del *scheduling* de React (Prioridad de Input vs. Prioridad de Renderizado)?
* **Conocimiento del Ecosistema del Navegador:** ¿Usaste correctamente Web Workers, WASM (conceptual) y el motor de renderizado de Canvas?
* **Pureza Arquitectónica:** ¿La solución es limpia, desacoplada y demuestra que el candidato no solo "usa" React, sino que "entiende" el *problema* que React resuelve y cuándo salirse de su camino?


Moraleja: Hay que leer primero. Todo esto es caspa, ni yo soy capaz de hacerlo (bueno si, pero no en 3 horas), se puede ir pa su casa ya