# 4 · Configuración y Despliegue

## 4.1 Estrategia de Despliegue (Cloud-Native)

El microservicio `vg-ms-paymentservice` está diseñado para operar en un entorno efímero en la nube. No requiere instalación local de servidores para su paso a producción. 

- **Plataforma de Hosting:** Render (Web Service).
- **Conectividad:** Despliegue automatizado/manual enlazado directamente a la rama principal del repositorio en GitHub/GitLab.
- **Protocolo:** Exposición segura y automática mediante HTTPS (`https://vg-ms-paymentservice.onrender.com`).

## 4.2 Configuración del Entorno (application.yml)

La configuración base del microservicio centraliza las credenciales, rutas externas y políticas de inicialización. En un entorno productivo, los valores sensibles (contraseñas) deben inyectarse como Variables de Entorno en Render.

```yaml
server:
  port: 8086

spring:
  # Configuración de Seguridad Delegada
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: [https://lab.vallegrande.edu.pe/keycloak-security/realms/prs-realm](https://lab.vallegrande.edu.pe/keycloak-security/realms/prs-realm)

  # Conexión Reactiva a Base de Datos Cloud
  r2dbc:
    url: r2dbc:postgresql://ep-sweet-cherry-ac1jujjy-pooler.sa-east-1.aws.neon.tech/paymentsservice?sslmode=require
    username: neondb_owner
    password: [SECRET]

  # Política de Inicialización de Esquemas
  sql:
    init:
      mode: always