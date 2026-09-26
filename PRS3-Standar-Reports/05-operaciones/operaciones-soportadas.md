# 5 · Operaciones Soportadas y Endpoints REST

Todas las operaciones del microservicio `vg-ms-paymentservice` están diseñadas bajo el estándar RESTful, procesadas de forma reactiva (`Mono`/`Flux`) y protegidas perimetralmente. 

Para consumir cualquier endpoint, el cliente debe incluir un encabezado HTTP de autorización válido:
`Authorization: Bearer <JWT_TOKEN>`

## 5.1 Gestión Transaccional de Pagos

El controlador principal (`PaymentRest.java`) expone las rutas base bajo el prefijo `/payments`.

| Método | Ruta REST | Descripción | Acceso / Roles |
|--------|-----------|-------------|----------------|
| **GET** | `/payments` | Lista todos los pagos registrados en el sistema. Los datos retornados incluyen campos enriquecidos de otros microservicios (Books, People, Requests). | Autenticado (Cualquier rol válido) |
| **POST** | `/payments` | Registra una nueva transacción de pago. Requiere un payload JSON estricto con los datos del monto, método de pago e items involucrados. Dispara bloques `@Transactional`. | Autenticado |
| **GET** | `/payments/{id}` | Recupera el detalle completo de una transacción específica a través de su identificador único. | Autenticado |

## 5.2 Generación de Reportes Financieros

A diferencia de otros módulos donde el reporte es 100% *client-side*, el microservicio de pagos puede exponer endpoints dedicados a la consolidación de datos sensibles para auditoría, protegidos por validación estricta de roles.

| Método | Ruta REST | Descripción | Acceso / Roles |
|--------|-----------|-------------|----------------|
| **GET** | `/payments/reportes/consolidado` | Obtiene la data consolidada de ingresos y transacciones en un periodo definido. | `@PreAuthorize("hasRole('ADMIN')")` |
| **GET** | `/payments/reportes/auditoria` | Retorna el registro de pagos anulados o marcados con inconsistencias para revisión de administradores. | `@PreAuthorize("hasRole('ADMIN')")` |

*(Nota: Los endpoints de reportes requieren que el token JWT contenga explícitamente el claim del rol `ADMIN` asignado desde Keycloak; de lo contrario, el servidor responderá con un error `403 Forbidden`).*

## 5.3 Documentación Dinámica (OpenAPI / Swagger)

Para facilitar la integración con el equipo de Frontend y asegurar que el contrato de la API esté siempre actualizado, el microservicio expone su propia documentación interactiva autogenerada mediante Springdoc OpenAPI.

Estas rutas son de acceso público (solo lectura de la documentación) y se configuran desde el `application.yml`:

| Ruta REST | Formato / Interfaz | Descripción |
|-----------|--------------------|-------------|
| `/api-docs` | JSON | Definición cruda del esquema OpenAPI 3.0 con todos los esquemas de petición y respuesta. |
| `/swagger-ui.html` | Interfaz Web | Consola visual e interactiva de Swagger UI para explorar y probar los endpoints manualmente. |

## 5.4 Flujo de Respuestas y Manejo de Errores

Cada endpoint garantiza devolver una respuesta estandarizada:

*   **Peticiones Exitosas (2xx):** Retornan directamente el objeto JSON (`Payment` o `List<Payment>`). Si una consulta de lista está vacía, se retorna HTTP 200 con un arreglo vacío `[]`.
*   **Excepciones de Negocio (4xx):** Respuestas con estructura de error detallando el motivo (ej. saldo insuficiente, referencia duplicada).
*   **Excepciones de Servidor (5xx):** Capturadas globalmente por interceptores para no exponer trazas de código fuente (Stacktraces) al cliente frontend, devolviendo únicamente el `status`, `error`, `path` y `timestamp`.