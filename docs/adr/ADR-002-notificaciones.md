# ADR-002: Envío asíncrono de notificaciones mediante cola de mensajes

- Estado: Aceptado
- Fecha: 28/10/2026
- Autor: Sebastian Carrera

## Contexto

SportBook debe enviar notificaciones de confirmación y recordatorio por correo electrónico y WhatsApp (RF5). Estos envíos dependen de servicios externos (proveedor de correo y API de mensajería) que pueden ser lentos, estar caídos o limitar la cantidad de mensajes por segundo.
 
La reserva con pago (RF3) es la operación más crítica del sistema y se concentra en fines de semana y festivos (atributo de disponibilidad). Un problema con el proveedor de correo o de WhatsApp no debe impedir ni demorar que un usuario complete su reserva.
 
De acuerdo con el ADR-001, el sistema es un monolito modular en Spring Boot; la decisión debe respetar esa arquitectura, la falta de equipo de operaciones y el presupuesto limitado.
 
Supuestos explícitos:
 
- Una notificación puede llegar unos segundos después de confirmada la reserva sin afectar al usuario.
- Los recordatorios se envían un tiempo fijo antes de la reserva (por ejemplo, 2 horas).
- El proveedor de nube ofrece un broker de mensajes administrado.
## Alternativas consideradas
 
1. **Alternativa A: Envío síncrono.** Al confirmar la reserva, el módulo de Reservas llama directamente al módulo de Notificaciones, que invoca al proveedor de correo y a la API de WhatsApp antes de responder al usuario.
   - Ventajas: implementación más simple; sin componentes adicionales; el resultado del envío se conoce de inmediato.
   - Desventajas: la respuesta al usuario se demora lo que tarden los proveedores externos; si un proveedor falla o no responde, la reserva puede quedar con error aunque el pago se haya hecho; no hay reintentos automáticos; en los picos de fin de semana las llamadas lentas ocupan hilos del servidor y degradan todo el sistema.
2. **Alternativa B: Cola de mensajes con broker (RabbitMQ administrado).** Al confirmar la reserva, el módulo de Reservas publica un evento `ReservaConfirmada` en una cola. El módulo de Notificaciones consume la cola en segundo plano y realiza los envíos.
   - Ventajas: la reserva responde rápido y no depende de los proveedores externos; reintentos automáticos y cola de mensajes fallidos , absorbe picos de mensajes; deja a Notificaciones desacoplado y listo para separarse como microservicio en el futuro.
   - Desventajas: un componente más que configurar, monitorear y pagar; consistencia eventual (la notificación llega segundos después); depuración más compleja; posibilidad de mensajes duplicados que deben manejarse.
3. **Alternativa C: Tabla de pendientes en la base de datos (*outbox*) con tarea programada.** La reserva guarda la notificación pendiente en una tabla dentro de la misma transacción, y una tarea programada (`@Scheduled`) la lee cada pocos segundos y la envía.
   - Ventajas: no requiere broker; la notificación se guarda en la misma transacción que la reserva, así que nunca se pierde.
   - Desventajas: consultas periódicas que cargan la base de datos única; mayor retraso; hay que programar a mano reintentos y control de concurrencia entre varias instancias; escala peor.
## Decisión
 
Se adopta la **Alternativa B**: comunicación **asíncrona** mediante un broker **RabbitMQ administrado** en la nube, usando **Spring AMQP**.
 
Funcionamiento:
 
- Cuando la reserva y el pago quedan confirmados, el módulo de Reservas publica el evento `ReservaConfirmada` después de que la transacción se guarda (`@TransactionalEventListener` en fase `AFTER_COMMIT`).
- El módulo de Notificaciones consume el evento y envía el correo (proveedor de correo vía API HTTPS) y el mensaje de WhatsApp (API de mensajería vía HTTPS).
- Una tarea programada publica eventos `RecordatorioReserva` para las reservas próximas.
- Si un envío falla, se reintenta con espera creciente; tras varios intentos, el mensaje pasa a una *dead-letter queue* para revisión.
- Cada mensaje lleva un identificador único para evitar enviar la misma notificación dos veces.
Justificación principal: protege la operación crítica (reserva y pago) frente a fallas y lentitud de servicios externos, mejora la disponibilidad en los picos y prepara la extracción futura de Notificaciones como microservicio, con un costo operativo bajo al usar un servicio administrado.
 
## Consecuencias
 
- **Positivas:**
  - La reserva no se demora ni falla por problemas del proveedor de correo o de WhatsApp.
  - Reintentos automáticos y registro de mensajes fallidos.
  - La cola absorbe picos de notificaciones los fines de semana.
  - Notificaciones queda desacoplado del resto, coherente con el monolito modular del ADR-001 y su evolución.
- **Negativas / riesgos:**
  - Se agrega un componente de infraestructura con costo mensual y que debe monitorearse.
  - Consistencia eventual: la notificación llega unos segundos después de la reserva.
  - Si la aplicación falla justo después de guardar la reserva y antes de publicar el evento, la notificación podría perderse; si se vuelve un problema, se combina con el patrón *outbox* (Alternativa C).
  - Posibles mensajes duplicados; se mitiga con el identificador único de cada mensaje.
  - Depurar un flujo asíncrono es más difícil; se mitiga con registros que incluyan el identificador de la reserva.
