# 4 · Configuración y Despliegue

## 4.1 Configuración de Entorno

- **Configuración externalizada:** El ecosistema FIDEI NEXUS utiliza variables de entorno inyectadas mediante el archivo `docker-compose.yml` (`environment:`) o archivos `.env` centralizados.
- **Base de datos por servicio:** Cada microservicio transaccional (`vg-ms-paymentService`, `vg-ms-booksService`) apunta a su propia base de datos PostgreSQL Serverless alojada en **Neon Tech**, accediendo de manera no bloqueante a través de la URI de R2DBC (`r2dbc:postgresql://...?sslmode=require`).
- **Delegación de Identidad OIDC:** El Gateway y los microservicios validan los tokens de acceso a través del Realm configurado en **Keycloak**.
- **Descubrimiento de Servicios:** Los microservicios resuelven sus dependencias internas utilizando nombres de host dentro de la red privada de Docker, inyectados vía variables como `SERVICES_PAYMENTS_URL` o `SERVICES_BOOKS_URL`.

## 4.2 Rutas del API Gateway (Dominios Equipo 3)

Dado que la arquitectura establece que los reportes se generan 100% en el frontend (Client-Side), el Gateway no expone rutas exclusivas para exportación de archivos. El cliente consume la información pura mediante los siguientes prefijos RESTful:

| Prefijo Enrutado | Servicio Destino | Puerto Interno | Dominio de Datos |
|------------------|------------------|----------------|------------------|
| `/api/v1/payments/**` | `vg-ms-paymentService` | `8082` | Transacciones financieras, comprobantes y consolidado de pagos. |
| `/api/v1/books/**` | `vg-ms-booksService` | `8081` | Catálogo de libros, control de inventario y stock. |

> **Nota de Seguridad:** El API Gateway estampa información de auditoría extraída del JWT y limpia las cabeceras entrantes maliciosas antes de enrutar la petición hacia los microservicios del backend, previniendo suplantación de identidad.

## 4.3 Dockerfile Genérico (Backend)

Para garantizar que los microservicios del Equipo 3 sean inmutables y portables, se utiliza un modelo de construcción multi-etapa (Multi-stage build) basado en Maven.

```dockerfile
# Etapa 1: Build de la aplicación Spring Boot WebFlux
FROM maven:3.9.5-eclipse-temurin-21 AS build
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline
COPY src ./src
RUN mvn clean package -DskipTests

# Etapa 2: Imagen de ejecución en producción
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY --from=build /app/target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]

```
## 4.4 Matriz de Despliegue en la Nube (Cloud Deployment)

Todos los microservicios del ecosistema FIDEI NEXUS y la aplicación cliente están estructurados en contenedores listos para su despliegue continuo (CI/CD) en plataformas PaaS o VPS. La siguiente matriz cubre todos los dominios del proyecto (PRS3):

| Componente del Sistema | Entorno / Plataforma Recomendada | Motor de Persistencia |
|------------------------|----------------------------------|-----------------------|
| `prs-eureka-server` | Render / VPS Valle Grande | — |
| `vg-ms-gateway` | Render / VPS Valle Grande | — |
| `vg-ms-auth` | Render / VPS Valle Grande | PostgreSQL |
| `vg-ms-booksService` | Render / VPS Valle Grande | PostgreSQL (Neon Tech) |
| `vg-ms-paymentService` | Render / VPS Valle Grande | PostgreSQL (Neon Tech) |
| `vg-ms-peopleService` | Render / VPS Valle Grande | PostgreSQL |
| `vg-ms-requestsService`| Render / VPS Valle Grande | PostgreSQL |
| `Frontend FIDEI NEXUS` | Vercel / Netlify / Firebase | — |

## 4.5 Configuración de CORS y Seguridad Perimetral

El único punto de acceso público expuesto a la red para todo el ecosistema es el API Gateway (`:9000`). Toda la comunicación interna entre los distintos microservicios (Pagos, Libros, Personas, Solicitudes) ocurre exclusivamente dentro de la malla interna aislada (`vg-network`). 

El Gateway es el único responsable de centralizar y configurar las políticas de Origen Cruzado (CORS) para permitir el acceso exclusivo desde las interfaces cliente autorizadas del PRS:

```yaml
CORS_ALLOWED_ORIGINS: >
  http://localhost:4200,
  http://localhost:5173,
  [https://fideinexus.vallegrande.edu.pe](https://fideinexus.vallegrande.edu.pe)




