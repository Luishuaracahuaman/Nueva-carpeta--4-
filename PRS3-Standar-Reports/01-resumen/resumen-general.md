# 1 · Resumen General

**Equipo 3 — Estándares y Forma de Trabajo: Módulo de Pagos (Backend)**

## Resumen Ejecutivo

El Equipo 3 es responsable del diseño, desarrollo y despliegue del dominio financiero dentro de la arquitectura orientada a microservicios del ecosistema **FIDEI NEXUS**. A diferencia de los módulos de consulta o catálogos, el microservicio de pagos (`vg-ms-paymentservice`) posee una naturaleza estrictamente **transaccional**. 

El diseño arquitectónico prioriza la consistencia de los datos, la atomicidad de las operaciones financieras y la alta disponibilidad. Toda la lógica de validación, procesamiento y persistencia se ejecuta aislada en el backend, garantizando que el estado financiero del sistema sea inmutable ante fallos del cliente o interrupciones de red.

## Alcance del Estándar

- **Plataforma Core:** FIDEI NEXUS — Ecosistema cloud-native.
- **Microservicio Asignado:** `vg-ms-paymentservice`.
- **Naturaleza del Componente:** API REST transaccional y segura.
- **Stack Tecnológico Base:** Spring Boot 3.x, Spring WebFlux, Java 21.

## Principios Arquitectónicos del Equipo

1. **Paradigma Reactivo y Concurrencia:** Implementación de un modelo de entrada/salida no bloqueante (NIO) utilizando Project Reactor (`Mono`/`Flux`) y drivers de base de datos reactivos (R2DBC). Esto maximiza el rendimiento y la tolerancia a picos de carga.
2. **Cumplimiento ACID y Transaccionalidad:** Aplicación estricta de límites transaccionales (`@Transactional`). Cualquier anomalía durante la orquestación de un pago desencadena un *rollback* automático, previniendo estados financieros huérfanos o inconsistentes.
3. **Seguridad Zero-Trust y Delegación de Identidad:** El microservicio opera bajo un modelo *stateless*. La autenticación y autorización se delegan completamente a un proveedor de identidad externo (Keycloak) mediante la validación criptográfica de tokens JWT (OAuth2 Resource Server).
4. **Infraestructura Cloud-Native y Efímera:** Adopción del patrón *Database-per-Service* alojado en la nube (Neon Tech Serverless Postgres). El contenedor de la aplicación se despliega en Render, garantizando que la infraestructura sea inmutable, escalable e independiente del entorno de desarrollo local.

## Códigos de Estado HTTP Semánticos

Para garantizar un estándar de comunicación uniforme entre el API Gateway, el Frontend y todos los microservicios del ecosistema FIDEI NEXUS, se establece el uso estricto de los siguientes códigos HTTP:

| Código HTTP | Escenario Estándar | Acción / Significado en el Backend |
|-------------|--------------------|------------------------------------|
| **200 OK** | Lectura o actualización exitosa | La petición se procesó correctamente. En consultas (GET), retorna el objeto o lista (una lista vacía retorna `[]` con 200, nunca 404). |
| **201 Created** | Escritura exitosa | Se creó y persistió un nuevo recurso en la base de datos de forma exitosa tras un POST. |
| **400 Bad Request** | Error de validación | El payload JSON enviado por el cliente tiene un formato incorrecto, faltan campos obligatorios o incumple reglas de negocio. |
| **401 Unauthorized**| Falta de autenticación | El token JWT Bearer está ausente en la cabecera, es inválido o ha expirado. |
| **403 Forbidden** | Permisos insuficientes | El token JWT es válido, pero el usuario autenticado no posee los roles (`Claims`) necesarios para ejecutar la acción. |
| **404 Not Found** | Recurso inexistente | El identificador (ID) o la ruta solicitada no existe en la base de datos del microservicio. |
| **500 Internal Error**| Falla crítica del servidor | Ocurrió un error inesperado (ej. pérdida de conexión a base de datos o excepción no controlada). La traza técnica se oculta por seguridad. |
