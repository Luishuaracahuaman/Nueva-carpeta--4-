# 6 · Información Adicional — Frontend FIDEI NEXUS (Web)

## 6.1 Stack Tecnológico del Cliente

El ecosistema FIDEI NEXUS cuenta con una interfaz de usuario moderna diseñada para integrarse fluidamente con los microservicios del backend.

- **Framework Principal:** Angular (Standalone Components) / Svelte.
- **Estilos y UI:** Tailwind CSS para un diseño responsivo y utilitario.
- **Comunicación HTTP:** Uso de interceptores para inyectar automáticamente el Bearer Token (JWT) en cada petición dirigida a `vg-ms-paymentservice`.
- **Gestión de Estado:** Manejo reactivo de las respuestas y flujos de carga durante el procesamiento de pagos.

## 6.2 Módulo Transaccional de Pagos

El frontend cuenta con vistas específicas para interactuar con la API REST de pagos, priorizando la validación de formularios y la retroalimentación al usuario.

| Vista / Componente | Acción Principal | Endpoint Consumido |
|--------------------|------------------|--------------------|
| **Listado de Pagos** | Muestra el historial de transacciones. Aprovecha los datos enriquecidos (`PeopleInfo`, `BookSaleInfo`) devueltos por el backend para mostrar nombres y títulos en lugar de simples IDs. | `GET /payments` |
| **Pasarela / Registro** | Formulario de validación estricta para registrar un nuevo pago. | `POST /payments` |
| **Panel de Auditoría** | Vista administrativa para verificar consolidación de ingresos. | `GET /payments/reportes/...` |

## 6.3 Modelos Transaccionales (TypeScript)

Para mantener la consistencia con el backend, el frontend define interfaces TypeScript que coinciden exactamente con el contrato JSON expuesto por el microservicio.

**Modelo de Envío (Request):**
```typescript
export interface PaymentRequest {
  tenantId: number;
  peopleId: number;
  requestId: number;
  amount: number;
  paymentMethod: string;
  reference: string;
  paymentDate: string; // ISO 8601
  status: string;
  items: BookItemRequest[];
}

export interface BookItemRequest {
  bookId: number;
  quantity: number;
}