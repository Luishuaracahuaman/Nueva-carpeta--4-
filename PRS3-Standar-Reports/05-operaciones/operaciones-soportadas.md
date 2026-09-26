# 5 · Operaciones Soportadas y Matriz de Endpoints

Todos los endpoints del backend en el **Equipo 3 (FIDEI NEXUS)** se sirven de forma **reactiva** (`Mono`/`Flux` de Project Reactor). Las operaciones de exportación y generación de reportes no existen como rutas en el backend; son procesos exclusivos del frontend (Angular/Svelte) que consumen los payloads JSON estándar.

## 5.1 Reportes Generados en el Cliente (Frontend)

Los botones de exportación se encuentran en las interfaces de usuario correspondientes. El cliente solicita los datos en formato JSON y utiliza las librerías `jsPDF` o `SheetJS` para construir el documento en memoria.

### Módulo de Pagos (`vg-ms-paymentService`)

| Acción en Interfaz | Servicio Angular / Función | Endpoint Consumido (JSON) | Salida |
|--------------------|----------------------------|---------------------------|--------|
| Descargar Comprobante | `exportarComprobantePdf(id)` | `GET /api/v1/payments/{id}/full` | `.pdf` (Voucher/A4) |
| Exportar Ingresos | `exportarConsolidadoExcel()` | `GET /api/v1/payments` | `.xlsx` |

### Módulo de Libros (`vg-ms-booksService`)

| Acción en Interfaz | Servicio Angular / Función | Endpoint Consumido (JSON) | Salida |
|--------------------|----------------------------|---------------------------|--------|
| Catálogo de Libros | `descargarCatalogoPdf()` | `GET /api/v1/books?status=DISPONIBLE`| `.pdf` (Apaisado) |
| Inventario / Stock | `exportarInventarioExcel()`| `GET /api/v1/books` | `.xlsx` |

## 5.2 Matriz de Endpoints REST (Fuente de Datos)

A continuación, se detalla el contrato de operaciones expuestas por los microservicios del Equipo 3 a través del API Gateway. Todos los endpoints requieren autenticación (Bearer JWT) y retornan `application/json`.

### Dominio de Pagos — `/api/v1/payments`

| Método | Ruta Específica | Descripción de la Operación |
|--------|-----------------|-----------------------------|
| `POST` | `/` | Registra una nueva transacción de pago. |
| `GET` | `/` | Lista el historial de transacciones (paginado). |
| `GET` | `/{id}` | Obtiene los datos crudos de un pago específico. |
| `GET` | `/{id}/full` | Obtiene el pago enriquecido (resuelve datos de persona y libro vía `WebClient`). |
| `PATCH`| `/{id}/status` | Actualiza el estado del pago (ej. `COMPLETADO`, `ANULADO`). |
| `GET` | `/customer/{peopleId}`| Historial de compras de un cliente específico. |

### Dominio de Libros — `/api/v1/books`

| Método | Ruta Específica | Descripción de la Operación |
|--------|-----------------|-----------------------------|
| `POST` | `/` | Registra un nuevo libro en el catálogo. |
| `PUT` | `/{id}` | Actualiza la información completa de un libro. |
| `GET` | `/` | Lista el catálogo (soporta filtros por categoría o estado). |
| `GET` | `/{id}` | Obtiene el detalle de un libro específico. |
| `PATCH`| `/{id}/stock` | Incrementa o reduce el stock disponible (operación transaccional). |
| `DELETE`| `/{id}` | Baja lógica del libro (cambia estado a `INACTIVO`). |

## 5.3 Integración Inter-servicio (WebClient)

Para construir la respuesta enriquecida del endpoint `GET /api/v1/payments/{id}/full` necesaria para imprimir el Comprobante en PDF, el microservicio de pagos actúa como orquestador y se comunica asíncronamente con otros dominios:

| Microservicio Origen | Llama a Microservicio Destino | Propósito en el Reporte / Documento |
|----------------------|-------------------------------|-------------------------------------|
| `vg-ms-paymentService` | `vg-ms-peopleService` | Obtener nombres, apellidos y DNI del cliente para la cabecera del comprobante. |
| `vg-ms-paymentService` | `vg-ms-booksService` | Obtener el título, autor y precio unitario del ítem adquirido. |

## 5.4 Protocolo de Códigos HTTP (Transaccional)

Las operaciones respetan estrictamente la semántica HTTP definida en el capítulo 2:

*   **200 OK:** Lectura exitosa de catálogos o transacciones.
*   **201 Created:** Pago registrado exitosamente (retorna el ID generado) o Libro creado.
*   **400 Bad Request:** Payload inválido (ej. intento de pago con monto negativo).
*   **403 Forbidden:** Intento de modificación de stock sin el rol de `INVENTORY_ADMIN`.
*   **404 Not Found:** ID de libro o comprobante inexistente.
*   **409 Conflict:** Intento de compra de un libro sin stock disponible.
*   **503 Service Unavailable:** Falla de resiliencia (ej. `vg-ms-peopleService` no responde al intentar generar el pago enriquecido).
