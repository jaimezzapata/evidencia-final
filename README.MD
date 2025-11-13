# Prueba Técnica React Avanzada: Explorador Interdimensional

## 1. El Desafío

Has sido contactado por una organización que monitorea el multiverso. Necesitan una interfaz de alta eficiencia para explorar datos de diferentes dimensiones, pero su API actual (basada en la API pública de Rick and Morty) tiene particularidades que requieren un manejo experto.

Tu misión es construir un "Explorador Interdimensional" que consuma esta API, manejando estados complejos, optimizando el rendimiento y gestionando peticiones dependientes.

**API Base:** `https://rickandmortyapi.com/api`

## 2. Requisitos Funcionales

### RF-01: Galería de Personajes (El Hub)

La vista principal (`/`) debe ser una galería de todos los personajes.

* **Paginación del Lado del Servidor:** La API (`/character`) devuelve datos paginados (20 ítems por página). Debes implementar una paginación funcional (botones "Siguiente" y "Anterior") que consuma los campos `info.next` e `info.prev` de la respuesta de la API.
* **Filtrado en Vivo:** El usuario debe poder filtrar la galería por:
    * Nombre (`name`).
    * Estado (`status`: Alive, Dead, Unknown).
* **Debouncing:** La aplicación **no debe** enviar una petición a la API por cada tecla pulsada en el filtro de nombre. Debes implementar un *debouncer* (ej. 300-500ms) para las peticiones de filtrado.
* **Estado de Carga y Error:** La UI debe reflejar claramente cuándo se están cargando datos (ej. un *skeleton loader* o *spinner*) y manejar/mostrar elegantemente cualquier error de la API (ej. si un filtro no devuelve resultados o la API falla).

### RF-02: Vista de Detalle (El Expediente)

Al hacer clic en la tarjeta de un personaje en la galería, el usuario debe ser navegado a una vista de detalle (ej. `/character/:id`).

* Esta vista debe mostrar la información básica del personaje obtenida de `/character/:id`.
* **Peticiones Dependientes (El Reto):** La respuesta de `/character/:id` **no** incluye la información completa de la ubicación o los episodios; solo proporciona URLs:
    1.  **Ubicación y Origen:** El objeto del personaje tiene `origin.url` y `location.url`. Debes realizar peticiones adicionales a estas URLs para obtener y mostrar los **nombres** de la ubicación y origen, no solo la URL.
    2.  **Lista de Episodios:** El objeto tiene un array `episode` que contiene múltiples URLs (una por episodio en el que aparece). Debes realizar peticiones a **todas** estas URLs (sugerencia: `Promise.all` o similar) y mostrar una lista con los nombres de dichos episodios.

### RF-03: Optimización y Manejo de Estado (Requisito Avanzado)

Este es el núcleo de la evaluación. No se aceptará el uso de `useState` + `useEffect` simple para el *data fetching*.

* **Gestor de Estado Asíncrono:** Debes usar una biblioteca moderna para la gestión del estado del servidor, como **TanStack Query (React Query)** o **SWR**.
* **Caching:** La aplicación debe cachear las respuestas de la API.
    * Si el usuario navega a la vista de detalle de un personaje (ej. Personaje A) y luego regresa a la galería, la galería no debe volver a cargarse (debe usar la caché).
    * Si el usuario vuelve a visitar la página del Personaje A, los datos deben cargarse instantáneamente desde la caché mientras se revalidan en segundo plano (*stale-while-revalidate*).
* **Virtualización (Opcional, pero muy valorado):** Si la galería principal tuviera miles de resultados, el DOM se volvería lento. Demuestra tu conocimiento de optimización implementando **virtualización de listas** (ej. `react-window` o `tanstack-virtual`) en la galería principal.

## 3. Requisitos Técnicos Obligatorios

* **Framework:** React (usando Hooks).
* **TypeScript:** El proyecto debe estar escrito en TypeScript.
* **Enrutamiento:** React Router (v6+).
* **Gestión de Estado de API:** TanStack Query (React Query) o SWR.
* **Estilos:** CSS Modules, Styled Components, Tailwind CSS, o cualquier solución moderna de CSS-in-JS. (La calidad visual no es el foco principal, pero la estructura CSS sí).
* **Pruebas (Testing):** Debes incluir al menos pruebas unitarias/integración (con Jest/RTL) para:
    1.  Un *custom hook* (si creas alguno).
    2.  El flujo de filtrado de la galería.
    3.  El manejo de las peticiones dependientes en la vista de detalle.

## 4. Criterios de Evaluación

* **Calidad y Arquitectura del Código:** Limpieza, modularidad (separación de componentes, lógica de negocio, servicios de API) y uso de TypeScript.
* **Manejo de Estado Asíncrono:** Uso correcto de la biblioteca de *data fetching* (manejo de `isLoading`, `isError`, `data`, `isFetching`).
* **Rendimiento:** Implementación correcta de *caching*, *debouncing* y (si se hizo) *virtualización*.
* **Manejo de Peticiones Complejas:** Resolución correcta de las peticiones dependientes y paralelas (RF-02).
* **Testing:** Cobertura y calidad de las pruebas.
* **Control de Código Fuente:** Claridad en los *commits* de Git.

## 5. Entregable

Debes entregar la URL de un repositorio de Git (GitHub, GitLab) con el código fuente completo e instrucciones claras en el `README.md` sobre cómo instalar dependencias y ejecutar el proyecto y las pruebas.


Moraleja: Esto es caspa, se puede ir pa su casa ya