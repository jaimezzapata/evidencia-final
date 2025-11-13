# Desafío de Implementación: El Reactor de Renderizado "Kairós" (Fases Mandatorias)

**Objetivo:** Construir un motor de visualización de datos de ultra-alta frecuencia (U-HFT) en React.

**Contexto:** Estás construyendo un visualizador para un Libro de Órdenes L3. Recibes decenas de miles de deltas por segundo vía WebSocket (formato binario) y debes renderizarlos a 60fps estables. El DOM no es una opción.

**Instrucción:** Debes ejecutar las siguientes fases en el orden y con las restricciones especificadas. El fracaso en cualquiera de las restricciones invalida la prueba.

---

### Fase 1: El Núcleo de Datos (Aislamiento del Hilo Principal)

El hilo principal (Main Thread) de JavaScript es un recurso sagrado reservado **exclusivamente** para la entrada del usuario y la confirmación final del renderizado. Todo el procesamiento de datos es asíncrono y paralelo.

1.  **Paso 1.1: Aislar la Red.**
    * Instancia un **Web Worker** dedicado (`network.worker.js`).
    * Este *worker* es el **único** responsable de establecer y mantener la conexión WebSocket.
    * Queda estrictamente prohibido instanciar el WebSocket en el hilo principal o en un componente de React.

2.  **Paso 1.2: Aislar el Cómputo (WASM).**
    * El *stream* del WebSocket emite datos binarios (simula un `ArrayBuffer`). JavaScript es demasiado lento para decodificar y calcular los deltas.
    * El Worker debe cargar un módulo **WebAssembly (WASM)** (puedes simularlo con un *stub* de Rust/Go/C++).
    * Este módulo WASM es el único responsable de:
        a.  Mantener el estado "actual" del libro de órdenes en su propia memoria.
        b.  Recibir el nuevo `ArrayBuffer` del WebSocket.
        c.  Calcular el *delta* (cambios) contra su estado anterior.
        d.  Generar una **estructura de datos de UI mínima** (ej. los 100 precios superiores/inferiores y sus volúmenes agregados).

3.  **Paso 1.3: El Puente de Datos.**
    * El Worker (`network.worker.js`), tras recibir el *snapshot* de UI desde WASM, envía este *snapshot* mínimo al hilo principal usando `postMessage()`.

---

### Fase 2: El Modelo de Estado (El Anti-React)

El estado de alta frecuencia **no debe** vivir en React. `useState` y `useReducer` están prohibidos para los datos del libro de órdenes, ya que su *batching* y *scheduling* no son controlables a este nivel.

1.  **Paso 2.1: Crear el Almacén Externo (External Store).**
    * En el hilo principal, crea un *store* de vainilla JS (un simple objeto o clase).
    * Este *store* debe exponer dos métodos:
        a.  `getSnapshot()`: Devuelve el *snapshot* actual del libro de órdenes.
        b.  `subscribe(callback)`: Añade un *listener* que será llamado cuando el estado cambie.

2.  **Paso 2.2: Actualizar el Almacén.**
    * El hilo principal debe registrar un *listener* (`onmessage`) para los mensajes del Worker (Fase 1.3).
    * Cuando se recibe un nuevo *snapshot* del Worker, este *listener* actualiza el *store* externo (Paso 2.1) e invoca a todos los *callbacks* suscritos.

3.  **Paso 2.3: El Puente Reactivo (`useSyncExternalStore`).**
    * Dentro de tu componente React, **debes** usar el *hook* **`useSyncExternalStore`** para suscribirte al almacén externo (Paso 2.1).
    * Esta es la **única** forma permitida de conectar el estado de alta frecuencia con el *scheduler* de React, garantizando que no haya *tearing* (desgarro de UI).

---

### Fase 3: El Motor de Renderizado (El Reconciliador Personalizado)

El DOM está prohibido. El Virtual DOM de React es demasiado lento. Debes implementar tu propio motor de renderizado de React que pinte directamente en `<canvas>`.

1.  **Paso 3.1: Instanciar el Reconciliador.**
    * Importa el paquete `react-reconciler`.
    * Debes crear tu propia instancia de reconciliador.

2.  **Paso 3.2: Implementar el "HostConfig".**
    * Aquí reside el núcleo del desafío. Debes implementar los métodos del "HostConfig" que le dicen a React cómo interactuar con tu "plataforma" (el Canvas).
    * Debes implementar, como mínimo:
        * `createInstance(type, props)`: Cuando React ve `<orderRow price={...}>`, este método debe crear un objeto de "instancia" (ej. `{ type: 'row', props, ... }`).
        * `createTextInstance()`: (Probablemente no sea necesario si todo es gráfico).
        * `appendInitialChild(parent, child)`: Para construir el árbol de renderizado.
        * `finalizeInitialChildren()`: (Ídem).
        * `commitUpdate(instance, updatePayload, type, oldProps, newProps)`: **El más crítico.** Aquí es donde React te dice "las *props* de esta instancia han cambiado". Es tu trabajo calcular el *delta* y ejecutar el comando de dibujo imperativo (ej. `ctx.fillRect`).
        * `appendChild(parent, child)`, `removeChild(parent, child)`, etc.

3.  **Paso 3.3: Crear los Componentes "Nativos" de Canvas.**
    * Tu componente principal (`<L3OrderBook />`) **no debe** devolver `<div>`.
    * Debe devolver tus propios elementos "primitivos" que tu reconciliador entiende, por ejemplo:
        ```jsx
        // Esto es lo que devuelve tu componente de React
        <canvasContainer>
          <orderBookView>
            {bids.map(bid => (
              <orderRow 
                key={bid.price}
                price={bid.price} 
                volume={bid.volume}
                type="bid"
              />
            ))}
          </orderBookView>
        </canvasContainer>
        ```

4.  **Paso 3.4: El Punto de Montaje (Root).**
    * Renderiza un único `<canvas>` en tu aplicación React normal (el "Shell").
    * Usa el método `ReactReconciler.createContainer()` para apuntar tu reconciliador personalizado a un objeto que *gestione* el contexto 2D de ese `<canvas>`.
    * Usa `ReactReconciler.updateContainer()` para "renderizar" tu componente `<L3OrderBook />` en tu motor personalizado.

---

### Fase 4: La Prueba de Concurrencia (La Validación)

Probar que el sistema funciona concurrentemente es mandatorio.

1.  **Paso 4.1: El Overlay de DOM.**
    * Renderiza un `<input type="text">` HTML estándar (DOM real) *encima* de tu `<canvas>` (usando `z-index`).

2.  **Paso 4.2: La Simulación.**
    * Inicia tu *mock* del WebSocket (Fase 1) para que envíe 20,000 actualizaciones por segundo.

3.  **Paso 4.3: La Prueba de Fuego.**
    * Mientras el `<canvas>` se está renderizando a 60fps (recibiendo miles de actualizaciones de estado vía `useSyncExternalStore` y tu reconciliador), **escribe en el `<input>`**.

4.  **Paso 4.4: El Criterio de Fracaso.**
    * Si el texto en el `<input>` muestra el más mínimo *lag*, *jank* (salto), o retraso, has fallado la prueba. Significa que tu motor de renderizado (Fase 3) o tu pipeline de datos (Fase 1) está bloqueando el hilo principal, impidiendo que React priorice la entrada del usuario (de alta prioridad) sobre el renderizado de datos (de baja prioridad).

---

### Fase 5: El Entregable

1.  **Código Fuente:** El repositorio con la implementación completa de las Fases 1-4.
2.  **Documento de Arquitectura (`ARCHITECTURE.md`):** Una justificación detallada de cada elección, explicando por qué `useSyncExternalStore` es superior a `useEffect` para este caso, cómo tu `HostConfig` maneja el *batching* de dibujos en el canvas, y cómo tu arquitectura previene el bloqueo del hilo principal.
3.  **Perfil de Rendimiento:** Capturas de pantalla o un video de la pestaña **Performance** de Chrome DevTools demostrando 60fps, un hilo principal libre (sin bloques rojos), y el trabajo de *scripting* pesado ocurriendo en el *Worker Thread*.


Moraleja: Hay que leer primero. Todo esto es caspa, ni yo soy capaz de hacerlo (bueno si, pero no en 3 horas), se puede ir pa su casa ya