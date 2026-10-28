# SportBook Cali — Arquitectura de Software

**Universidad de San Buenaventura Cali** · Ingeniería de Sistemas · Arquitectura de Software

**Docente:** Daniel Pareja

**Estudiante:** Sebastian Carrera

## Parte 1. Marco conceptual

### Tabla comparativa: MVC, SOA y microservicios

| Criterio | MVC | SOA | Microservicios |
|---|---|---|---|
| **Acoplamiento** | Alto entre funcionalidades: todo vive en una sola aplicación y comparte código y base de datos. Las capas M, V y C sí están separadas entre sí. | Medio: los servicios se separan mediante contratos (p. ej. SOAP/WSDL), pero dependen del bus de servicios (ESB) y suelen compartir modelos de datos. | Bajo: cada servicio es independiente, se comunica por APIs ligeras (REST) o eventos y no comparte base de datos con otros. |
| **Despliegue** | Una sola unidad (un `.jar` o `.war`). Cualquier cambio obliga a volver a desplegar toda la aplicación. | Los servicios se despliegan por separado, pero de forma coordinada con el ESB; despliegues pesados y planificados. | Independiente por servicio, frecuente y automatizado (CI/CD, contenedores). |
| **Escalabilidad** | Se escala la aplicación completa: más recursos al servidor (vertical) o más copias detrás de un balanceador (horizontal). No se puede escalar solo una parte. | Por servicio, aunque el ESB puede convertirse en cuello de botella. | Granular: se escala solo el servicio con más demanda. |
| **Complejidad operativa** | Baja: una aplicación, una base de datos y un solo pipeline de despliegue. | Alta: hay que administrar el ESB, los contratos y el gobierno de los servicios. | Muy alta: muchos servicios, contenedores, orquestación, monitoreo y trazas distribuidas, y manejo de fallas de red. |
| **Gestión de datos** | Una base de datos centralizada con transacciones ACID simples. | Datos frecuentemente compartidos entre servicios o bajo un modelo canónico empresarial. | Base de datos por servicio; consistencia eventual y transacciones distribuidas mediante patrones como Saga. |
| **Tamaño de equipo ideal** | Pequeño (aprox. 1–10 desarrolladores) trabajando como un solo equipo. | Grande, empresarial, con varios equipos y un área de integración. | Varios equipos pequeños y autónomos, cada uno dueño de sus servicios, con capacidades de DevOps. |
| **Cuándo conviene** | Productos nuevos o MVP, dominio acotado, equipo pequeño y plazos cortos. | Integrar sistemas heterogéneos o heredados en organizaciones grandes. | Sistemas grandes cuyas partes escalan y evolucionan a ritmos distintos, en organizaciones con madurez en DevOps. |



## Parte 4. Reflexión critica 

**1. ¿Cuál es el mayor riesgo de la arquitectura elegida?**

El mayor riesgo es que el monolito modular se degrade en un monolito acoplado: con la presión de las entregas, los módulos pueden empezar a usar directamente las tablas de otros, y la evolución planeada hacia microservicios se volvería costosa. Además, la aplicación y la base de datos única comparten el mismo punto de falla: un error grave o una consulta lenta en los picos de fin de semana afectaría a todo el sistema, incluidas las reservas y los pagos. Se mitiga con pruebas de arquitectura (Spring Modulith o ArchUnit), varias instancias y monitoreo.

**2. ¿Qué cambiaría con diez veces el tráfico planeado?**

Primero aliviaría la base de datos con caché (Redis) y réplicas de lectura para la consulta de disponibilidad, que es la operación más frecuente. Luego extraería como microservicios los módulos con más carga, empezando por Disponibilidad y Notificaciones (este ya está desacoplado por la cola), cada uno con su propia base de datos y escalado independiente, siguiendo el patrón strangler. Esto exigiría contenedores con orquestación administrada, monitoreo distribuido y al menos una persona de DevOps.

---

## Declaración de uso de inteligencia artificial

Usé Claude (Anthropic, modelo Opus 5.5) como apoyo para: explicar los conceptos de MVC, SOA, microservicios, ADR y modelo C4; proponer la redacción de la tabla comparativa, los ADR-001 y ADR-002 y la reflexión crítica.
