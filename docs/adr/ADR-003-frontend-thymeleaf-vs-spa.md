# ADR-003: Interfaz web con Thymeleaf en lugar de una SPA con React

- Estado: Aceptado
- Fecha: 28/10/2026
- Autor: Sebastian Carrera

## Contexto

SportBook necesita una interfaz web para jugadores (RF1, RF2, RF3, RF6) y para administradores de complejos (RF4). El equipo es de 5 desarrolladores con experiencia en Java y Spring Boot, sin experiencia declarada en frontend, y el primer lanzamiento debe estar listo en 4 meses. Este ADR contrasta una decisión descartada: construir el frontend como una aplicación separada (SPA).

Supuesto explícito: en la primera versión no se requiere aplicación móvil nativa; una web adaptable a celulares es suficiente.

## Alternativas consideradas

1. Alternativa A (descartada): SPA con React que consume una API REST de Spring Boot.
   - Ventajas: experiencia de usuario más fluida; la misma API serviría a una futura app móvil; frontend y backend se despliegan por separado.
   - Desventajas: el equipo debe aprender React y JavaScript/TypeScript; hay dos proyectos, dos pipelines y un contenedor más que operar; hay que diseñar y asegurar una API pública (CORS, tokens); alto riesgo para el plazo de 4 meses.

2. Alternativa B (elegida): páginas generadas en el servidor con Thymeleaf dentro del monolito Spring Boot.
   - Ventajas: usa solo Java y Spring, que el equipo ya domina; un solo proyecto y un solo despliegue, coherente con el ADR-001; es directamente la Vista del patrón MVC; seguridad más simple con sesiones de Spring Security.
   - Desventajas: interfaz menos dinámica; cada acción recarga la página o parte de ella; si luego se crea una app móvil, habrá que construir una API REST.

## Decisión

Se usa Thymeleaf en la Aplicación web SportBook. Se descarta la SPA con React porque agrega un lenguaje, un proyecto y un despliegue más, en contra de las restricciones de equipo, tiempo y operación. Para mejorar la interactividad en pantallas puntuales, como la consulta de disponibilidad, se pueden usar pequeños fragmentos de JavaScript.

## Consecuencias

- Positivas: menor tiempo de desarrollo; un solo despliegue; coherencia con el monolito modular y con MVC; menos superficie de seguridad.
- Negativas / riesgos: experiencia de usuario menos fluida que una SPA; si el negocio exige app móvil, habrá que exponer una API REST y posiblemente revertir parcialmente esta decisión (estado "Reemplazado").
