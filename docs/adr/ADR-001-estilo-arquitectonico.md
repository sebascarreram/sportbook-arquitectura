# ADR-001: Monolito modular como estilo arquitectónico de SportBook

- Estado: Aceptado
- Fecha: 28/10/2026
- Autor: Sebastian Carrera

## Contexto

SportBook es una plataforma nueva para reservar canchas deportivas (fútbol 5, pádel, baloncesto) en complejos de Cali. Debe cubrir registro y perfil de usuarios (RF1), consulta de disponibilidad (RF2), reservas con pago en línea mediante una pasarela externa (RF3), gestión de canchas, horarios y precios por parte de cada complejo (RF4), notificaciones por correo y WhatsApp (RF5) y calificaciones (RF6).

Atributos de calidad y restricciones que influyen en la decisión:

- **Tiempo de entrega:** el primer lanzamiento debe estar listo en 4 meses.
- **Equipo:** 5 desarrolladores con experiencia en Java y Spring Boot, sin equipo de operaciones dedicado.
- **Presupuesto:** limitado; se prioriza infraestructura en la nube con pago por uso.
- **Disponibilidad:** la demanda se concentra en fines de semana y festivos.
- **Escalabilidad:** se espera pasar de 5 a 60 complejos deportivos en 18 meses.

Supuestos explícitos:

- Al inicio, la carga máxima esperada es de cientos de usuarios concurrentes en los picos de fin de semana, no de miles, es un crecimiento moderado.
- SportBook no debe integrarse con sistemas heredados propios, solo con servicios externos (pasarela de pagos, proveedor de correo y API de mensajería).
- Se usará un proveedor de nube con servicios administrados (base de datos, balanceador, autoescalado y colas).

## Alternativas consideradas

1. **Alternativa A: Microservicios.** Un servicio independiente por dominio (usuarios, disponibilidad, reservas, pagos, notificaciones, calificaciones), cada uno con su propia base de datos y comunicándose por REST o eventos.
   - Ventajas: escalado granular (por ejemplo, solo disponibilidad en los picos), despliegues independientes, fallas aisladas entre servicios.
   - Desventajas: complejidad operativa muy alta (contenedores, orquestación, monitoreo y trazas distribuidas) sin equipo de operaciones; transacciones distribuidas entre reserva y pago (patrón Saga, consistencia eventual); mayor costo de infraestructura; con 5 desarrolladores no hay equipos autónomos que aprovechen la independencia; alto riesgo de no cumplir los 4 meses.

2. **Alternativa B: SOA con bus de servicios (ESB).** Servicios de negocio de grano grueso comunicados a través de un ESB central.
   - Ventajas: servicios reutilizables y separados por contratos; buena para integrar muchos sistemas.
   - Desventajas: el ESB es un componente costoso y complejo de operar, y un punto único de falla y posible cuello de botella; su fortaleza es integrar sistemas heredados, que SportBook no tiene.

3. **Alternativa C: Monolito modular (MVC por capas).** Una sola aplicación Spring Boot, desplegada como una unidad, dividida en módulos por dominio con límites claros y organizada internamente en capas (controlador, servicio, repositorio).
   - Ventajas: baja complejidad operativa; una sola base de datos con transacciones ACID para reserva y pago, aprovecha la experiencia del equipo en Spring Boot, desarrollo y pruebas más rápidos, menor costo.
   - Desventajas: escalado de toda la aplicación y no por partes, un error grave puede afectar a todo el sistema; cada cambio requiere redesplegar la aplicación completa; riesgo de que los módulos se acoplen con el tiempo si no se respetan los límites.

## Decisión

Se adopta un **monolito modular** (Alternativa C) en Spring Boot, con estos módulos: **Usuarios**, **Complejos y Disponibilidad**, **Reservas y Pagos**, **Notificaciones** y **Calificaciones**.

Reglas de diseño:

- Cada módulo expone una interfaz pública (servicios Java) y solo se comunica con otros módulos por esa interfaz; ningún módulo accede directamente a las tablas de otro.
- Una sola base de datos PostgreSQL, con un esquema por módulo.
- La aplicación se ejecuta en varias instancias detrás de un balanceador de carga con autoescalado en la nube para atender los picos de fines de semana y festivos.
- Las notificaciones se envían de forma asíncrona (ver ADR-002).

Justificación principal: las restricciones de tiempo (4 meses), equipo (5 desarrolladores sin operaciones) y presupuesto pesan más que la necesidad de escalado granular. El crecimiento de 5 a 60 complejos se puede atender replicando la aplicación. Los límites claros entre módulos permiten extraer microservicios más adelante si un módulo lo necesita.

## Consecuencias

- **Positivas:**
  - Mayor probabilidad de cumplir el lanzamiento en 4 meses.
  - Operación simple y económica: un artefacto, un pipeline de despliegue, una base de datos administrada.
  - Consistencia fuerte en reserva y pago gracias a transacciones locales.
  - Depuración y pruebas más sencillas.
  - Camino de evolución definido hacia microservicios (patrón *strangler*), empezando por Disponibilidad o Notificaciones.
- **Negativas / riesgos:**
  - No se puede escalar un módulo de forma independiente, en los picos se escala toda la aplicación, con algo de desperdicio de recursos.
  - Un fallo grave (por ejemplo, falta de memoria) puede afectar a todos los módulos al mismo tiempo; se mitiga con varias instancias y verificaciones de salud.
  - La base de datos única puede convertirse en cuello de botella, se mitiga con índices, réplicas de lectura y caché para la consulta de disponibilidad.
  - Riesgo de erosión de los límites entre módulos; se mitiga con revisiones de código y pruebas de arquitectura.
