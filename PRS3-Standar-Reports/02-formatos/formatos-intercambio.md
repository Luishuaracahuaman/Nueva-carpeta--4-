# 2 · Formatos de Intercambio de Datos y Estándares API

A diferencia de las arquitecturas monolíticas o los módulos frontend encargados de exportar documentos (PDF/Excel), los microservicios del backend en el ecosistema **FIDEI NEXUS** operan exclusivamente como proveedores y consumidores de APIs RESTful. Por lo tanto, el único formato estándar y oficial de intercambio de información es **JSON (JavaScript Object Notation)**.

## 2.1 Tecnologías de Serialización y Mapeo

Para garantizar una comunicación eficiente y reactiva entre el cliente (Frontend), el API Gateway y los microservicios internos, el Equipo 3 estandariza el uso de las siguientes herramientas:

| Capa de Aplicación | Tecnología / Librería | Propósito Estándar |
|--------------------|-----------------------|--------------------|
| **Capa Web (REST)** | Jackson (`spring-boot-starter-json`) | Serialización y deserialización automática de payloads HTTP. |
| **Integración Interna** | `WebClient` (Spring WebFlux) | Consumo asíncrono y no bloqueante de APIs entre microservicios. |
| **Capa de Persistencia**| Spring Data R2DBC | Mapeo reactivo entre objetos Java y registros en bases de datos relacionales (PostgreSQL). |

## 2.2 Estándares de Tipos de Datos y Formateo

Para mantener la consistencia de los datos en todos los dominios (`vg-ms-booksService` y `vg-ms-paymentService`), se establecen las siguientes reglas obligatorias de tipado:

1. **Datos Financieros y Monetarios:** 
   * **Regla Estricta:** Queda prohibido el uso de `Double` o `Float` para cálculos financieros debido a la pérdida de precisión por coma flotante.
   * **Estándar:** Utilizar exclusivamente `java.math.BigDecimal` para atributos como precios, montos de pago e impuestos.
2. **Fechas y Marcas de Tiempo (Timestamps):**
   * **Formato de Transmisión:** Todas las fechas en los JSON de entrada y salida deben respetar el estándar **ISO 8601** (`YYYY-MM-DDTHH:mm:ss`).
   * **Mapeo en Backend:** Utilizar `java.time.LocalDateTime` o `java.time.LocalDate`.
3. **Optimización de Payloads (Control de Nulos):**
   * Se debe implementar la anotación `@JsonInclude(JsonInclude.Include.NON_NULL)` a nivel de clase (DTOs y Modelos) para evitar la transmisión de atributos vacíos, optimizando el ancho de banda de la red.

## 2.3 Estructura de Contratos JSON (Payloads)

Los microservicios deben definir contratos claros separando los datos físicos (base de datos) de los datos enriquecidos (transitorios). 

### Ejemplo: Dominio de Libros (`vg-ms-booksService`)

El servicio expone la información del catálogo. El payload de respuesta omite metadatos internos e incluye la estructura de autores y categorías.

```json
{
  "id": 5,
  "title": "Arquitectura Cloud-Native",
  "isbn": "978-3-16-148410-0",
  "price": 45.50,
  "stock": 120,
  "category": "Tecnología",
  "status": "DISPONIBLE"
}

```
### Ejemplo: Dominio de Pagos (`vg-ms-paymentService`)

El servicio requiere un payload de entrada (POST) estricto para procesar la transacción. Los campos autogenerados (`id`, `createdAt`) no se envían en la petición.

```json
{
  "tenantId": 1,
  "peopleId": 101,
  "amount": 150.50,
  "paymentMethod": "TRANSFERENCIA",
  "reference": "VOUCHER-9988",
  "paymentDate": "2026-09-25T10:30:00",
  "items": [
    {
      "bookId": 5,
      "quantity": 2
    }
  ]
}
```
## 2.4 Códigos de Estado HTTP Semánticos

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

