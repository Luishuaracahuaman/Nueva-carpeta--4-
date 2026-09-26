# 2 · Formatos de Intercambio de Datos y Estándares API

A diferencia de los módulos orientados a la exportación de documentos (PDF/Excel), el microservicio transaccional `vg-ms-paymentservice` opera exclusivamente como un proveedor y consumidor de APIs REST. Por lo tanto, el formato estándar de intercambio de información es **JSON (JavaScript Object Notation)**.

## 2.1 Tecnologías de Serialización y Mapeo

| Tipo de Intercambio | Tecnología / Librería | Uso Específico |
|---------------------|-----------------------|----------------|
| **Capa Web (REST)** | Jackson (`spring-boot-starter-json`) | Serialización/Deserialización de payloads HTTP. |
| **Capa de Persistencia** | Spring Data R2DBC | Mapeo reactivo entre objetos Java y registros PostgreSQL. |
| **Integración Externa** | WebClient (Spring WebFlux) | Consumo asíncrono de APIs externas (Books, People, Requests) esperando respuestas en JSON. |

## 2.2 Estándares para Tipos de Datos Críticos (Financieros)

Para garantizar la integridad y precisión de las transacciones, el equipo establece las siguientes reglas obligatorias de tipado:

1. **Manejo de Divisas y Montos:** 
   * **Prohibido:** El uso de `Double` o `Float`.
   * **Estándar:** Utilizar exclusivamente `java.math.BigDecimal` para los campos financieros (ej. `amount`) para evitar errores de precisión por coma flotante durante los cálculos o la persistencia.
2. **Estándar de Fechas y Marcas de Tiempo (Timestamps):**
   * **Formato:** Todas las fechas deben transmitirse en formato **ISO 8601** (`YYYY-MM-DDTHH:mm:ss`).
   * **Backend:** Mapeo mediante `java.time.LocalDateTime` (ej. `paymentDate`, `createdAt`, `updatedAt`).
3. **Control de Nulos en Respuestas:**
   * Uso de la anotación `@JsonInclude(JsonInclude.Include.NON_NULL)` a nivel de clase para optimizar el ancho de banda, evitando enviar atributos con valor nulo al cliente.

## 2.3 Estructura del Payload Transaccional (JSON)

Las peticiones de creación (POST) hacia el endpoint `/payments` deben respetar una estructura de contrato estricta. Los datos autogenerados por el sistema (como `id` o `createdAt`) se omiten en la solicitud y son inyectados por la base de datos.

### Ejemplo de Payload (Request - POST)
```json
{
  "tenantId": 1,
  "peopleId": 101,
  "requestId": 201,
  "amount": 150.50,
  "paymentMethod": "TRANSFERENCIA",
  "reference": "VOUCHER-9988",
  "paymentDate": "2026-09-25T10:30:00",
  "status": "COMPLETADO",
  "items": [
    {
      "bookId": 5,
      "quantity": 2
    }
  ]
}