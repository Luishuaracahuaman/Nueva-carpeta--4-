# 6 · Frontend FIDEI NEXUS (Client-Side)

## 6.1 Stack Tecnológico y Arquitectura UI

- **Frameworks:** Aplicación SPA desarrollada con **Angular** (Standalone Components, Signals) y componentes dinámicos en **Svelte**[cite: 19, 20, 21].
- **Estilos:** **Tailwind CSS** para un diseño responsivo, atómico y mantenible.
- **Comunicación:** Integración exclusiva contra el **API Gateway** (`http://localhost:9000` vía proxy en desarrollo) consumiendo datos estructurados en formato JSON.
- **Procesamiento de Reportes:** 100% Client-Side. El frontend no recibe archivos binarios ni Blobs desde el backend; asume toda la responsabilidad de transformar los DTOs en documentos finales descargables[cite: 19, 21].

## 6.2 Módulos con Capacidad de Exportación (Equipo 3)

Los botones de descarga están integrados directamente en las vistas principales y modales de detalle de cada dominio:

### Módulo Transaccional — `/pagos`

| Elemento | Detalle |
|----------|---------|
| Componente principal | `pages/pagos/historial-pagos.ts` |
| Servicio de reporte | `core/services/payments-report.service.ts` |
| Métodos de descarga | `exportarComprobantePdf()` · `exportarConsolidadoExcel()` |
| Estado | ✅ Implementado y funcional |

### Módulo de Catálogo — `/libros`

| Elemento | Detalle |
|----------|---------|
| Componente principal | `pages/libros/gestion-inventario.ts` |
| Servicio de reporte | `core/services/books-report.service.ts` |
| Métodos de descarga | `descargarCatalogoPdf()` · `exportarInventarioExcel()` |
| Estado | ✅ Implementado y funcional |

## 6.3 Estructura Modular de Servicios de Reporte

Para mantener la escalabilidad y aislar responsabilidades, los servicios de exportación se ubican en la capa `core` y se dividen por formato y dominio de negocio[cite: 20]:

```text
src/app/core/services/
├── payments/
│   ├── payment-pdf.service.ts       ← Renderizado de Voucher jsPDF
│   └── payment-excel.service.ts     ← Consolidado financiero SheetJS
└── books/
    ├── book-catalog-pdf.service.ts  ← Catálogo comercial jsPDF
    └── book-inventory-excel.service.ts ← Stock y métricas SheetJS
```
## 6.4 Modelos y Tipado Estricto (TypeScript)

Para garantizar que la renderización de las tablas y encabezados no falle en tiempo de ejecución, el frontend tipa estrictamente los JSON enriquecidos que provienen del backend:

**Archivo:** `core/models/payment-enriched.interface.ts`
```typescript
export interface PaymentEnrichedResponse {
  paymentId: number;
  amount: number;
  paymentMethod: string;
  date: string;
  customer: {
    id: number;
    name: string;
    documentNumber: string;
  };
  bookPurchased: {
    id: number;
    title: string;
    author: string;
    price: number;
  };
}
```

## 6.5 Reglas y Convenciones del Frontend

1. **Importación Perezosa (Lazy Loading):** Es estrictamente obligatorio usar importaciones dinámicas (`await import('jspdf')` y `await import('xlsx')`) dentro del método de generación[cite: 19, 20]. Nunca deben importarse a nivel global del archivo para no penalizar el tiempo de carga inicial de la aplicación (bundle size)[cite: 19].
2. **Independencia del Servidor:** Los blobs generados en memoria nunca se envían de vuelta al servidor; se descargan de forma directa y nativa en el navegador del usuario[cite: 19, 21].
3. **Deshabilitación Reactiva:** Los botones de exportación a Excel y PDF deben deshabilitarse (`[disabled]="!hayDatos"`) automáticamente cuando la lista de datos a exportar está vacía[cite: 19].
4. **Formateo de Datos:** Los valores provenientes de enums del backend (ej. `CREDIT_CARD`, `OUT_OF_STOCK`) deben formatearse mediante una función de utilidad (ej. `formatearTexto()`) antes de inyectarse al documento final para que sean legibles por el usuario final[cite: 19].



