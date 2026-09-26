# 3 · Arquitectura General

## 3.1 Stack Tecnológico del Módulo de Pagos

El microservicio `vg-ms-paymentservice` está diseñado bajo un modelo cloud-native y reactivo, asegurando alta disponibilidad y rendimiento en operaciones transaccionales.

- **Framework Core:** Spring Boot 3.x con Spring WebFlux (Java).
- **Persistencia Reactiva:** Driver R2DBC (`spring-boot-starter-data-r2dbc`) para operaciones no bloqueantes.
- **Base de Datos:** PostgreSQL Serverless alojado en **Neon Tech** (`ep-sweet-cherry-...`). No se utilizan bases de datos locales.
- **Seguridad:** Spring Security OAuth2 Resource Server validando tokens JWT emitidos por **Keycloak** (`prs-realm`).
- **Despliegue:** Contenedor Docker auto-gestionado y desplegado en la plataforma cloud **Render**.

## 3.2 Diagrama de Flujo Transaccional

A diferencia de los servicios de consulta, el registro de un pago requiere validación estricta y persistencia ACID. El siguiente diagrama ilustra el flujo de una petición de escritura (POST) exitosa:

```mermaid
sequenceDiagram
    participant C as Cliente (Frontend / Postman)
    participant MS as vg-ms-paymentservice (Render)
    participant KC as Keycloak (Identity Provider)
    participant DB as PostgreSQL (Neon Tech)

    C->>MS: POST /payments + Header: Bearer Token + JSON Payload
    MS->>KC: Valida firma y vigencia del JWT
    KC-->>MS: Token Válido (Claims del Usuario)
    MS->>MS: Inicia bloque @Transactional
    MS->>DB: INSERT INTO payments (r2dbc no bloqueante)
    DB-->>MS: Confirmación de persistencia (ID generado)
    MS->>MS: Finaliza bloque @Transactional (Commit)
    MS-->>C: HTTP 201 Created + JSON Resultante