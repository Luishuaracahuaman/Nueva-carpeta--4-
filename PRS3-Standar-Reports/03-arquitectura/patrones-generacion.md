# 4 · Patrones de Generación y Comunicación Reactiva

## 4.1 Patrón WebClient con Propagación de Token JWT (Backend)

En la arquitectura reactiva de FIDEI NEXUS, los microservicios del Equipo 3 (`vg-ms-paymentService` y `vg-ms-booksService`) se comunican entre sí utilizando `WebClient`. Para garantizar la trazabilidad y la seguridad transaccional, se inyecta un filtro que propaga el encabezado `Authorization: Bearer` recibido desde el API Gateway a las peticiones internas.

```java
@Configuration
public class WebClientConfiguration {

    @Bean
    public WebClient webClient(WebClient.Builder builder) {
        return builder
            .filter(passBearerTokenFilter())
            .build();
    }

    private ExchangeFilterFunction passBearerTokenFilter() {
        return (request, next) -> Mono.deferContextual(contextView -> {
            String token = contextView.getOrDefault("jwt_token", "");
            ClientRequest filteredRequest = ClientRequest.from(request)
                    .header(HttpHeaders.AUTHORIZATION, "Bearer " + token)
                    .build();
            return next.exchange(filteredRequest);
        });
    }
}

## 4.2 Patrón DTO Enriquecido Multiservicio (Backend)

Para generar comprobantes de pago o catálogos detallados, el microservicio orquestador combina información de múltiples dominios utilizando operadores reactivos (`Mono.zip` o `flatMap`). Esto evita que el frontend tenga que hacer múltiples peticiones HTTP separadas.

```java
public Mono<PaymentEnrichedResponse> getFullPaymentDetails(Long paymentId) {
    return paymentRepository.findById(paymentId)
        .flatMap(payment -> Mono.zip(
            peopleClient.getPersonById(payment.getPeopleId()),
            booksClient.getBookById(payment.getBookId())
        ).map(tuple -> {
            PersonDto person = tuple.getT1();
            BookDto book = tuple.getT2();

            return PaymentEnrichedResponse.builder()
                .paymentId(payment.getId())
                .amount(payment.getAmount())
                .paymentMethod(payment.getPaymentMethod())
                .customer(person)
                .bookPurchased(book)
                .build();
        }));
}


## 4.3 PDF de Comprobantes de Pago — Frontend (Client-Side)

El frontend asume la responsabilidad exclusiva de renderizar los documentos basándose en el JSON enriquecido devuelto por el backend.

### Estructura del Documento (Voucher / Ticket)

```text
┌─────────────────────────────────────────────┐
│  LOGO FIDEI NEXUS (Centrado)                │
│  "Comprobante de Pago Electrónico"          │
│  Referencia: {reference} | Fecha: {date}    │
├─────────────────────────────────────────────┤
│  DATOS DEL CLIENTE                          │
│  Nombre: {customer.name} | DNI: {customer.dni}
├─────────────────────────────────────────────┤
│  DETALLE DE LA TRANSACCIÓN (autoTable grid) │
│  Libro | Cantidad | Precio Unit. | Subtotal │
├─────────────────────────────────────────────┤
│  TOTAL A PAGAR: S/ {amount}                 │
│  Método de Pago: {paymentMethod}            │
└─────────────────────────────────────────────┘

## 4.4 Excel de Consolidado Financiero — Frontend (Client-Side)

### Estructura del Libro (Workbook)

```text
Workbook
├── Hoja "Transacciones"
│   Columnas: ID Pago | Fecha | Cliente | Libro | Monto | Método | Estado
│
└── Hoja "Resumen"
    Total de Ingresos | Libro Más Vendido | Filtros Aplicados
```

- **Librería:** `SheetJS` (`xlsx`).
- **API usada:** `XLSX.utils.json_to_sheet()` y `XLSX.writeFile()`.
- **Nombre de Archivo:** `Reporte_Financiero_{YYYY-MM-DD}.xlsx`.

## 4.5 Estrategia Lazy Import (Rendimiento)

Para evitar que el peso inicial (bundle size) de la aplicación cliente aumente innecesariamente, las librerías pesadas de exportación se cargan de forma dinámica (lazy load) solo cuando el usuario ejecuta la acción de descarga.

```typescript
// Método ejecutado desde el componente UI tras recibir el JSON del backend
async exportarComprobantePdf(data: PaymentEnrichedResponse): Promise<void> {
  
  // Importación dinámica (Lazy Load)
  const { default: jsPDF } = await import('jspdf');
  const { default: autoTable } = await import('jspdf-autotable');

  const doc = new jsPDF();
  
  // Configuración de Cabecera
  doc.setFontSize(16);
  doc.text('FIDEI NEXUS - Comprobante de Pago', 14, 15);
  doc.setFontSize(11);
  doc.text(`Cliente: ${data.customer.name}`, 14, 25);
  
  // Generación de Tabla de Items
  autoTable(doc, {
    startY: 35,
    head: [['Libro Adquirido', 'Método', 'Total']],
    body: [
      [data.bookPurchased.title, data.paymentMethod, `S/ ${data.amount}`]
    ]
  });

  // Descarga nativa en el navegador
  doc.save(`Comprobante_${data.paymentId}.pdf`);
}