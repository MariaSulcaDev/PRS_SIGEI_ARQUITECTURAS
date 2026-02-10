# 🏗️ ARQUITECTURA SIGEI - Sistema Integrado de Gestión Educativa Inicial

## Arquitectura de Microservicios con DDD, Arquitectura Hexagonal y Multi-Tenancy

---

## 📑 Índice

1. [Visión General](#1-visión-general)
2. [Stack Tecnológico](#2-stack-tecnológico)
3. [Arquitectura de Alto Nivel](#3-arquitectura-de-alto-nivel)
4. [Comunicación entre Microservicios](#4-comunicación-entre-microservicios)
5. [Patrón Multi-Tenant](#5-patrón-multi-tenant)
6. [Microservicios del Sistema](#6-microservicios-del-sistema)
7. [Arquitectura Hexagonal + DDD](#7-arquitectura-hexagonal--ddd)
8. [API Gateway](#8-api-gateway)
9. [Seguridad con Keycloak](#9-seguridad-con-keycloak)
10. [Mensajería con RabbitMQ](#10-mensajería-con-rabbitmq)
11. [Microservicio de Notificaciones](#11-microservicio-de-notificaciones)
12. [Estructura de Carpetas Estandarizada](#12-estructura-de-carpetas-estandarizada)
13. [Configuración por Variables de Entorno](#13-configuración-por-variables-de-entorno)
14. [Despliegue con Docker Compose](#14-despliegue-con-docker-compose)

---

## 1. Visión General

### 1.1 Descripción del Sistema

**SIGEI** es un sistema de gestión educativa diseñado específicamente para colegios de nivel inicial en Perú. El sistema es **multi-tenant**, permitiendo que múltiples instituciones educativas operen de forma aislada en la misma infraestructura.

### 1.2 Objetivos Arquitectónicos

| Objetivo | Descripción |
|----------|-------------|
| **Simplicidad** | Fácil de entender y mantener por el equipo |
| **Multi-Tenancy** | Aislamiento completo entre instituciones |
| **Mantenibilidad** | Código limpio, modular y testeable |
| **Escalabilidad** | Preparado para crecer cuando sea necesario |
| **Seguridad** | Autenticación/Autorización centralizada con Keycloak |

### 1.3 Principios de Diseño

- **Domain-Driven Design (DDD)**: Modelado basado en el dominio del negocio
- **Arquitectura Hexagonal**: Separación clara entre dominio e infraestructura
- **Comunicación Híbrida**: REST síncrono + RabbitMQ asíncrono
- **API-First**: Contratos de API documentados con OpenAPI
- **Configuración Simple**: Variables de entorno en Docker Compose

---

## 2. Stack Tecnológico

### 2.1 Versiones Estandarizadas

| Tecnología | Versión | Propósito |
|------------|---------|-----------|
| **Java** | 17 LTS | Lenguaje base |
| **Spring Boot** | 3.5.10 | Framework principal |
| **Spring WebFlux** | 3.5.10 | Programación reactiva |
| **Spring Data R2DBC** | 3.5.10 | Acceso reactivo a PostgreSQL |
| **Spring Data MongoDB** | 3.5.10 | Acceso reactivo a MongoDB |
| **Spring Security** | 6.x | Seguridad OAuth2/JWT |
| **Keycloak** | 24.x | Identity Provider |
| **PostgreSQL** | 16.x | Base de datos relacional |
| **MongoDB** | 7.x | Base de datos documental |
| **RabbitMQ** | 3.13.x | Mensajería asíncrona |
| **Docker** | 25.x | Containerización |
| **Docker Compose** | 2.x | Orquestación en VPC |

### 2.2 ¿Por qué esta selección?

| Decisión | Justificación |
|----------|---------------|
| **Sin Redis** | Keycloak maneja sesiones. Caché se puede agregar después si hay problemas de rendimiento |
| **Sin Eureka** | Docker Compose permite comunicación por nombre de contenedor directamente |
| **Sin Config Server** | Variables de entorno en Docker Compose son más simples y suficientes |
| **RabbitMQ vs Kafka** | RabbitMQ es más simple, menos curva de aprendizaje, ideal para equipos pequeños |
| **Gateway sí** | Único punto de entrada para el frontend, centraliza seguridad |

### 2.3 Dependencias Maven Estándar

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.5.10</version>
</parent>

<properties>
    <java.version>17</java.version>
    <springdoc.version>2.8.8</springdoc.version>
    <mapstruct.version>1.6.3</mapstruct.version>
</properties>

<dependencies>
    <!-- WebFlux (Reactivo) -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-webflux</artifactId>
    </dependency>

    <!-- Seguridad OAuth2 -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
    </dependency>

    <!-- RabbitMQ -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-amqp</artifactId>
    </dependency>

    <!-- OpenAPI/Swagger -->
    <dependency>
        <groupId>org.springdoc</groupId>
        <artifactId>springdoc-openapi-starter-webflux-ui</artifactId>
        <version>${springdoc.version}</version>
    </dependency>

    <!-- Lombok -->
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <optional>true</optional>
    </dependency>
</dependencies>
```

---

## 3. Arquitectura de Alto Nivel

### 3.1 Diagrama de Arquitectura Simplificada

```
                                    ┌─────────────────────────────────────────────────────────────┐
                                    │                        FRONTEND                             │
                                    │              React 19 + TypeScript + Vite                   │
                                    │                     sigei-web                               │
                                    │                    Puerto: 3000                             │
                                    └─────────────────────────┬───────────────────────────────────┘
                                                              │
                                                              ▼
                                    ┌─────────────────────────────────────────────────────────────┐
                                    │                     API GATEWAY                             │
                                    │               Spring Cloud Gateway                          │
                                    │       (Routing, Seguridad JWT, CORS, Rate Limit)           │
                                    │                    Puerto: 8080                             │
                                    └─────────────────────────┬───────────────────────────────────┘
                                                              │
                         ┌────────────────────────────────────┴────────────────────────────────────┐
                         │                                                                         │
                         ▼                                                                         ▼
          ┌──────────────────────────┐                                          ┌──────────────────────────┐
          │      KEYCLOAK            │                                          │    NGINX (Producción)   │
          │   Identity Provider      │                                          │    Reverse Proxy        │
          │   Puerto: 8180           │                                          │    Puerto: 80/443       │
          └──────────────────────────┘                                          └──────────────────────────┘
                                                              │
                                                              ▼
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                        MICROSERVICIOS DE NEGOCIO                                            │
│                           (Comunicación REST directa por nombre de contenedor Docker)                       │
│                                                                                                             │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐   │
│  │  ms-institution │  │   ms-users      │  │  ms-students    │  │  ms-academic    │  │ ms-enrollments  │   │
│  │    :9080        │  │    :9081        │  │    :9082        │  │    :9083        │  │    :9084        │   │
│  │    MongoDB      │  │    MongoDB      │  │    MongoDB      │  │   PostgreSQL    │  │   PostgreSQL    │   │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘  └─────────────────┘  └─────────────────┘   │
│                                                                                                             │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐   │
│  │  ms-attendance  │  │    ms-grades    │  │  ms-behavior    │  │  ms-psychology  │  │   ms-events     │   │
│  │    :9085        │  │    :9086        │  │    :9087        │  │    :9088        │  │    :9089        │   │
│  │   PostgreSQL    │  │   PostgreSQL    │  │   PostgreSQL    │  │   PostgreSQL    │  │   PostgreSQL    │   │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘  └─────────────────┘  └─────────────────┘   │
│                                                                                                             │
│  ┌─────────────────┐  ┌─────────────────┐                                                                  │
│  │ ms-teacher-     │  │ ms-notification │◀──────── Consume mensajes de RabbitMQ                           │
│  │  assignment     │  │    :9091        │          (Envía emails, push, SMS, WhatsApp a padres)           │
│  │    :9090        │  │    MongoDB      │                                                                  │
│  │   PostgreSQL    │  └─────────────────┘                                                                  │
│  └─────────────────┘                                                                                        │
└───────────────────────────────────────────────────────────────────────────────────┬─────────────────────────┘
                                                                                    │
                                                                                    ▼
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                             RABBITMQ                                                        │
│                                   (Solo para eventos asíncronos)                                            │
│                                  Puerto: 5672 (AMQP) / 15672 (UI)                                          │
│                                                                                                             │
│   Exchanges: attendance.events | document.events | notification.commands                                    │
│   Casos de uso: Notificar ausencia → Enviar a padres | Compartir documento → Enviar a padres              │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
                                                              │
                                                              ▼
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                           BASES DE DATOS                                                    │
│                                                                                                             │
│  ┌─────────────────────────────────────────┐  ┌─────────────────────────────────────────┐                  │
│  │           PostgreSQL :5432              │  │            MongoDB :27017               │                  │
│  │                                         │  │                                         │                  │
│  │ • ms-academic (cursos, competencias)    │  │ • ms-institution (instituciones, aulas) │                  │
│  │ • ms-enrollments (matrículas)           │  │ • ms-users (usuarios)                   │                  │
│  │ • ms-attendance (asistencias)           │  │ • ms-students (estudiantes, apoderados) │                  │
│  │ • ms-grades (notas, evaluaciones)       │  │ • ms-notification (logs, plantillas)    │                  │
│  │ • ms-behavior (comportamiento)          │  │                                         │                  │
│  │ • ms-psychology (bienestar)             │  │                                         │                  │
│  │ • ms-events (eventos, calendario)       │  │                                         │                  │
│  │ • ms-teacher-assignment (asignaciones)  │  │                                         │                  │
│  └─────────────────────────────────────────┘  └─────────────────────────────────────────┘                  │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
```

### 3.2 Resumen de Puertos

| Servicio | Puerto | Descripción |
|----------|--------|-------------|
| **Nginx** | 80/443 | Proxy reverso (producción) |
| **Gateway** | 8080 | API Gateway |
| **Keycloak** | 8180 | Identity Provider |
| **RabbitMQ** | 5672 | AMQP |
| **RabbitMQ UI** | 15672 | Administración |
| **PostgreSQL** | 5432 | Base de datos relacional |
| **MongoDB** | 27017 | Base de datos documental |
| **Frontend** | 3000 | React App |
| **Microservicios** | 9080-9091 | APIs de negocio |

---

## 4. Comunicación entre Microservicios

### 4.1 Modelo Híbrido: REST + RabbitMQ

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                          COMUNICACIÓN HÍBRIDA                                            │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                         │
│  ┌─────────────────────────────────────────────────────────────────────────────────┐   │
│  │                      SÍNCRONA (REST / WebClient)                                 │   │
│  │                                                                                  │   │
│  │  Usar cuando:                                                                    │   │
│  │  ✅ Necesitas respuesta INMEDIATA                                               │   │
│  │  ✅ Operaciones CRUD normales                                                   │   │
│  │  ✅ Consultas de datos                                                          │   │
│  │  ✅ Validaciones que requieren datos de otro servicio                           │   │
│  │                                                                                  │   │
│  │  Ejemplos:                                                                       │   │
│  │  • Gateway → ms-students: "Dame datos del estudiante X"                         │   │
│  │  • ms-enrollment → ms-students: "¿Existe este estudiante?"                      │   │
│  │  • ms-attendance → ms-students: "Obtener lista de estudiantes del aula"         │   │
│  └─────────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                         │
│  ┌─────────────────────────────────────────────────────────────────────────────────┐   │
│  │                      ASÍNCRONA (RabbitMQ)                                        │   │
│  │                                                                                  │   │
│  │  Usar cuando:                                                                    │   │
│  │  ✅ NO necesitas esperar respuesta                                              │   │
│  │  ✅ Tareas que pueden ejecutarse en background                                  │   │
│  │  ✅ Notificaciones a usuarios/padres                                            │   │
│  │  ✅ Generación de reportes                                                      │   │
│  │  ✅ Envío de emails, SMS, push notifications                                    │   │
│  │                                                                                  │   │
│  │  Ejemplos:                                                                       │   │
│  │  • ms-attendance → RabbitMQ → ms-notification: "Estudiante ausente, avisar"     │   │
│  │  • ms-grades → RabbitMQ → ms-notification: "Libreta lista, enviar a padre"      │   │
│  │  • ms-behavior → RabbitMQ → ms-notification: "Incidente registrado, notificar"  │   │
│  └─────────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

### 4.2 Comunicación REST entre Microservicios (Docker)

En Docker Compose, los contenedores se comunican por **nombre de servicio**:

```java
@Configuration
public class WebClientConfig {

    @Bean
    public WebClient studentsWebClient() {
        return WebClient.builder()
            // En Docker Compose usamos el nombre del contenedor
            .baseUrl("http://ms-students:9082")
            .build();
    }

    @Bean
    public WebClient institutionWebClient() {
        return WebClient.builder()
            .baseUrl("http://ms-institution:9080")
            .build();
    }
}

// Uso en un servicio
@Service
@RequiredArgsConstructor
public class EnrollmentService {

    private final WebClient studentsWebClient;

    public Mono<StudentResponse> getStudent(String studentId) {
        return studentsWebClient
            .get()
            .uri("/api/v1/students/{id}", studentId)
            .retrieve()
            .bodyToMono(StudentResponse.class);
    }
}
```

### 4.3 Flujo de Comunicación Típico

```
┌──────────┐     ┌─────────┐     ┌──────────────┐     ┌─────────────┐     ┌─────────────┐
│  Cliente │     │ Gateway │     │ Microservice │     │  Database   │     │  RabbitMQ   │
└────┬─────┘     └────┬────┘     └──────┬───────┘     └──────┬──────┘     └──────┬──────┘
     │                │                 │                    │                   │
     │ 1. Request     │                 │                    │                   │
     │───────────────▶│                 │                    │                   │
     │                │                 │                    │                   │
     │                │ 2. Validate JWT │                    │                   │
     │                │ (Keycloak)      │                    │                   │
     │                │─────────────────│                    │                   │
     │                │                 │                    │                   │
     │                │ 3. Forward      │                    │                   │
     │                │───────────────▶ │                    │                   │
     │                │                 │                    │                   │
     │                │                 │ 4. Query DB        │                   │
     │                │                 │───────────────────▶│                   │
     │                │                 │                    │                   │
     │                │                 │ 5. Response        │                   │
     │                │                 │◀───────────────────│                   │
     │                │                 │                    │                   │
     │                │                 │ 6. Publish Event   │                   │
     │                │                 │    (si aplica)     │                   │
     │                │                 │────────────────────────────────────────▶
     │                │                 │                    │                   │
     │ 7. Response    │                 │                    │                   │
     │◀───────────────│◀────────────────│                    │                   │
```

---

## 5. Patrón Multi-Tenant

### 5.1 Estrategia: Discriminator Column

Utilizamos una columna `tenant_id` en todas las tablas para aislar datos por institución:

```
┌─────────────────────────────────────────────────────────────────┐
│                    ESTRATEGIA: SHARED DATABASE                  │
│                    CON DISCRIMINATOR COLUMN                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    Base de Datos                         │   │
│  │                                                          │   │
│  │  ┌─────────────────────────────────────────────────┐    │   │
│  │  │              students                            │    │   │
│  │  ├─────────────────────────────────────────────────┤    │   │
│  │  │ id | tenant_id | first_name | last_name | ...   │    │   │
│  │  ├─────────────────────────────────────────────────┤    │   │
│  │  │ 1  | inst_001  | Juan       | Pérez     | ...   │    │   │
│  │  │ 2  | inst_001  | María      | García    | ...   │    │   │
│  │  │ 3  | inst_002  | Pedro      | López     | ...   │    │   │
│  │  │ 4  | inst_002  | Ana        | Torres    | ...   │    │   │
│  │  └─────────────────────────────────────────────────┘    │   │
│  │                                                          │   │
│  │  ⚠️ Cada query incluye automáticamente:                 │   │
│  │     WHERE tenant_id = :currentTenant                     │   │
│  │                                                          │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                 │
│  Ventajas:                                                      │
│  ✅ Menor costo de infraestructura (una sola BD)               │
│  ✅ Fácil mantenimiento                                         │
│  ✅ Adecuado para colegios pequeños/medianos                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 5.2 Extracción del Tenant desde JWT

El `tenant_id` se extrae del token JWT (claim `institution_id` de Keycloak):

```java
// TenantExtractor.java
@Component
public class TenantExtractor {

    public Mono<String> extractTenantId(ServerWebExchange exchange) {
        return ReactiveSecurityContextHolder.getContext()
            .map(ctx -> ctx.getAuthentication())
            .filter(auth -> auth instanceof JwtAuthenticationToken)
            .cast(JwtAuthenticationToken.class)
            .map(jwt -> jwt.getToken().getClaimAsString("institution_id"))
            .switchIfEmpty(Mono.error(new UnauthorizedException("No tenant found")));
    }
}

// TenantContext.java - Para WebFlux
public class TenantContext {
    public static final String TENANT_KEY = "tenantId";

    public static Function<Context, Context> withTenant(String tenantId) {
        return ctx -> ctx.put(TENANT_KEY, tenantId);
    }

    public static Mono<String> getTenantId() {
        return Mono.deferContextual(ctx ->
            Mono.just(ctx.getOrDefault(TENANT_KEY, ""))
        );
    }
}
```

### 5.3 Modelo base con Tenant

```java
// Para R2DBC (PostgreSQL)
public abstract class TenantAwareEntity {

    @Column("tenant_id")
    private String tenantId;

    // getter, setter
}

// Ejemplo de entidad
@Table("attendance")
public class Attendance extends TenantAwareEntity {

    @Id
    private UUID id;

    @Column("student_id")
    private String studentId;

    @Column("date")
    private LocalDate date;

    @Column("status")
    private AttendanceStatus status;
}
```

```java
// Para MongoDB
@Document(collection = "students")
public class Student {

    @Id
    private String id;

    @Field("tenant_id")
    @Indexed
    private String tenantId;

    @Field("first_name")
    private String firstName;

    // ... más campos
}
```

---

## 6. Microservicios del Sistema

### 6.1 Inventario de Microservicios

| Microservicio | Puerto | Base de Datos | Descripción |
|---------------|--------|---------------|-------------|
| **sigei-gateway** | 8080 | - | API Gateway, routing, seguridad |
| **ms-institution** | 9080 | MongoDB | Instituciones, aulas, turnos |
| **ms-users** | 9081 | MongoDB | Usuarios del sistema (docentes, admin) |
| **ms-students** | 9082 | MongoDB | Estudiantes y apoderados |
| **ms-academic** | 9083 | PostgreSQL | Cursos, competencias, capacidades |
| **ms-enrollments** | 9084 | PostgreSQL | Matrículas y períodos académicos |
| **ms-attendance** | 9085 | PostgreSQL | Control de asistencias |
| **ms-grades** | 9086 | PostgreSQL | Notas y evaluaciones |
| **ms-behavior** | 9087 | PostgreSQL | Comportamiento e incidentes |
| **ms-psychology** | 9088 | PostgreSQL | Bienestar psicológico |
| **ms-events** | 9089 | PostgreSQL | Eventos y calendario |
| **ms-teacher-assignment** | 9090 | PostgreSQL | Asignación de docentes |
| **ms-notification** | 9091 | MongoDB | Notificaciones (consume RabbitMQ) |

### 6.2 Bounded Contexts (DDD)

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                                    BOUNDED CONTEXTS                                      │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                         │
│  ┌─────────────────────────────────────────────────────────────────────────────────┐   │
│  │                           CORE DOMAIN (Dominios Principales)                     │   │
│  │                                                                                  │   │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐                  │   │
│  │  │   INSTITUTION   │  │    STUDENTS     │  │    ENROLLMENT   │                  │   │
│  │  │    CONTEXT      │  │    CONTEXT      │  │    CONTEXT      │                  │   │
│  │  ├─────────────────┤  ├─────────────────┤  ├─────────────────┤                  │   │
│  │  │ • Institution   │  │ • Student       │  │ • Enrollment    │                  │   │
│  │  │ • Classroom     │  │ • Guardian      │  │ • AcademicPeriod│                  │   │
│  │  │ • AcademicYear  │  │ • HealthInfo    │  │ • Document      │                  │   │
│  │  └─────────────────┘  └─────────────────┘  └─────────────────┘                  │   │
│  │                                                                                  │   │
│  │  ┌─────────────────┐  ┌─────────────────┐                                       │   │
│  │  │    ACADEMIC     │  │   ATTENDANCE    │                                       │   │
│  │  │    CONTEXT      │  │    CONTEXT      │                                       │   │
│  │  ├─────────────────┤  ├─────────────────┤                                       │   │
│  │  │ • Course        │  │ • Attendance    │                                       │   │
│  │  │ • Competency    │  │ • Justification │                                       │   │
│  │  │ • Capacity      │  │ • DailyReport   │                                       │   │
│  │  └─────────────────┘  └─────────────────┘                                       │   │
│  └─────────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                         │
│  ┌─────────────────────────────────────────────────────────────────────────────────┐   │
│  │                        SUPPORTING DOMAIN (Dominios de Soporte)                   │   │
│  │                                                                                  │   │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐                  │   │
│  │  │     GRADES      │  │    BEHAVIOR     │  │   PSYCHOLOGY    │                  │   │
│  │  │    CONTEXT      │  │    CONTEXT      │  │    CONTEXT      │                  │   │
│  │  ├─────────────────┤  ├─────────────────┤  ├─────────────────┤                  │   │
│  │  │ • Evaluation    │  │ • BehaviorRecord│  │ • PsyEvaluation │                  │   │
│  │  │ • ReportCard    │  │ • Incident      │  │ • SpecialNeeds  │                  │   │
│  │  │ • Competency    │  │ • FollowUp      │  │ • Support       │                  │   │
│  │  │   Achievement   │  │                 │  │                 │                  │   │
│  │  └─────────────────┘  └─────────────────┘  └─────────────────┘                  │   │
│  │                                                                                  │   │
│  │  ┌─────────────────┐  ┌─────────────────┐                                       │   │
│  │  │     EVENTS      │  │    TEACHER      │                                       │   │
│  │  │    CONTEXT      │  │   ASSIGNMENT    │                                       │   │
│  │  ├─────────────────┤  ├─────────────────┤                                       │   │
│  │  │ • Event         │  │ • Assignment    │                                       │   │
│  │  │ • Calendar      │  │ • Schedule      │                                       │   │
│  │  └─────────────────┘  └─────────────────┘                                       │   │
│  └─────────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                         │
│  ┌─────────────────────────────────────────────────────────────────────────────────┐   │
│  │                          GENERIC DOMAIN (Dominios Genéricos)                     │   │
│  │                                                                                  │   │
│  │  ┌─────────────────┐  ┌─────────────────┐                                       │   │
│  │  │     USERS       │  │  NOTIFICATION   │                                       │   │
│  │  │    CONTEXT      │  │    CONTEXT      │                                       │   │
│  │  ├─────────────────┤  ├─────────────────┤                                       │   │
│  │  │ • User          │  │ • Notification  │                                       │   │
│  │  │ • Role          │  │ • Template      │                                       │   │
│  │  │ • Permission    │  │ • Channel       │                                       │   │
│  │  └─────────────────┘  └─────────────────┘                                       │   │
│  └─────────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 7. Arquitectura Hexagonal + DDD

### 7.1 Capas de la Arquitectura

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                              ARQUITECTURA HEXAGONAL + DDD                                │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                         │
│  ┌─────────────────────────────────────────────────────────────────────────────────┐   │
│  │                            INFRASTRUCTURE LAYER                                  │   │
│  │                         (Adaptadores Primarios - Driving)                        │   │
│  │                                                                                  │   │
│  │  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐                        │   │
│  │  │     REST      │  │   RabbitMQ    │  │   Scheduled   │                        │   │
│  │  │  Controllers  │  │   Consumers   │  │     Tasks     │                        │   │
│  │  └───────────────┘  └───────────────┘  └───────────────┘                        │   │
│  └─────────────────────────────────────────────────────────────────────────────────┘   │
│                                           │                                             │
│                                           ▼                                             │
│  ┌─────────────────────────────────────────────────────────────────────────────────┐   │
│  │                              APPLICATION LAYER                                   │   │
│  │                           (Casos de Uso / Servicios)                             │   │
│  │                                                                                  │   │
│  │  ┌───────────────────────────────────────────────────────────────────────┐      │   │
│  │  │                         PORTS (Interfaces)                             │      │   │
│  │  │                                                                        │      │   │
│  │  │  ┌─────────────────────────┐    ┌─────────────────────────┐          │      │   │
│  │  │  │     Input Ports         │    │     Output Ports        │          │      │   │
│  │  │  │  (Use Case Interfaces)  │    │  (Repository Interfaces)│          │      │   │
│  │  │  └─────────────────────────┘    └─────────────────────────┘          │      │   │
│  │  └───────────────────────────────────────────────────────────────────────┘      │   │
│  │                                                                                  │   │
│  │  ┌───────────────────────────────────────────────────────────────────────┐      │   │
│  │  │                         USE CASES                                      │      │   │
│  │  │  CreateStudentUseCase | RegisterAttendanceUseCase | SendNotification  │      │   │
│  │  └───────────────────────────────────────────────────────────────────────┘      │   │
│  └─────────────────────────────────────────────────────────────────────────────────┘   │
│                                           │                                             │
│                                           ▼                                             │
│  ┌─────────────────────────────────────────────────────────────────────────────────┐   │
│  │                                DOMAIN LAYER                                      │   │
│  │                              (Núcleo del Negocio)                                │   │
│  │                                                                                  │   │
│  │  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐    │   │
│  │  │   ENTITIES    │  │ VALUE OBJECTS │  │DOMAIN EVENTS  │  │DOMAIN SERVICES│    │   │
│  │  │   Student     │  │   StudentId   │  │StudentCreated │  │   Policies    │    │   │
│  │  │   Attendance  │  │   Address     │  │AttendanceMarked│  │   Rules       │    │   │
│  │  └───────────────┘  └───────────────┘  └───────────────┘  └───────────────┘    │   │
│  └─────────────────────────────────────────────────────────────────────────────────┘   │
│                                           ▲                                             │
│                                           │                                             │
│  ┌─────────────────────────────────────────────────────────────────────────────────┐   │
│  │                            INFRASTRUCTURE LAYER                                  │   │
│  │                         (Adaptadores Secundarios - Driven)                       │   │
│  │                                                                                  │   │
│  │  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐    │   │
│  │  │  Repository   │  │   RabbitMQ    │  │  WebClient    │  │   External    │    │   │
│  │  │  Adapters     │  │   Publisher   │  │   (HTTP)      │  │   Services    │    │   │
│  │  │ (R2DBC/Mongo) │  │               │  │               │  │   (Email,SMS) │    │   │
│  │  └───────────────┘  └───────────────┘  └───────────────┘  └───────────────┘    │   │
│  └─────────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

### 7.2 Regla de Dependencias

```
                    ┌─────────────────────────────────────────┐
                    │                                         │
                    │   Las dependencias SIEMPRE apuntan      │
                    │   hacia el CENTRO (Domain Layer)        │
                    │                                         │
                    │   El Domain NO depende de nada externo  │
                    │                                         │
                    └─────────────────────────────────────────┘

                              Infrastructure
                                    │
                                    ▼
                              Application
                                    │
                                    ▼
                                 Domain     ◀── Núcleo puro, sin dependencias externas
                                    ▲
                                    │
                              Application
                                    ▲
                                    │
                              Infrastructure
```

---

## 8. API Gateway

### 8.1 Configuración del Gateway

```yaml
# application.yml del Gateway
server:
  port: 8080

spring:
  application:
    name: sigei-gateway

  cloud:
    gateway:
      routes:
        # Instituciones
        - id: ms-institution
          uri: http://ms-institution:9080
          predicates:
            - Path=/api/v1/institutions/**, /api/v1/classrooms/**

        # Usuarios
        - id: ms-users
          uri: http://ms-users:9081
          predicates:
            - Path=/api/v1/users/**

        # Estudiantes
        - id: ms-students
          uri: http://ms-students:9082
          predicates:
            - Path=/api/v1/students/**, /api/v1/guardians/**

        # Académico
        - id: ms-academic
          uri: http://ms-academic:9083
          predicates:
            - Path=/api/v1/courses/**, /api/v1/competencies/**

        # Matrículas
        - id: ms-enrollments
          uri: http://ms-enrollments:9084
          predicates:
            - Path=/api/v1/enrollments/**, /api/v1/academic-periods/**

        # Asistencias
        - id: ms-attendance
          uri: http://ms-attendance:9085
          predicates:
            - Path=/api/v1/attendance/**

        # Notas
        - id: ms-grades
          uri: http://ms-grades:9086
          predicates:
            - Path=/api/v1/grades/**, /api/v1/evaluations/**

        # Comportamiento
        - id: ms-behavior
          uri: http://ms-behavior:9087
          predicates:
            - Path=/api/v1/behavior/**, /api/v1/incidents/**

        # Psicología
        - id: ms-psychology
          uri: http://ms-psychology:9088
          predicates:
            - Path=/api/v1/psychology/**

        # Eventos
        - id: ms-events
          uri: http://ms-events:9089
          predicates:
            - Path=/api/v1/events/**, /api/v1/calendar/**

        # Asignación docente
        - id: ms-teacher-assignment
          uri: http://ms-teacher-assignment:9090
          predicates:
            - Path=/api/v1/teacher-assignments/**

        # Notificaciones
        - id: ms-notification
          uri: http://ms-notification:9091
          predicates:
            - Path=/api/v1/notifications/**

  # Seguridad con OAuth2/Keycloak
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: ${KEYCLOAK_ISSUER_URI:http://keycloak:8080/realms/sigei}
          jwk-set-uri: ${KEYCLOAK_JWK_URI:http://keycloak:8080/realms/sigei/protocol/openid-connect/certs}
```

### 8.2 Seguridad en Gateway

```java
@Configuration
@EnableWebFluxSecurity
public class GatewaySecurityConfig {

    @Bean
    public SecurityWebFilterChain securityWebFilterChain(ServerHttpSecurity http) {
        return http
            .csrf(csrf -> csrf.disable())
            .cors(cors -> cors.configurationSource(corsConfigurationSource()))
            .authorizeExchange(exchanges -> exchanges
                // Endpoints públicos
                .pathMatchers("/actuator/health").permitAll()
                .pathMatchers("/swagger-ui/**", "/v3/api-docs/**").permitAll()

                // Endpoints protegidos por rol
                .pathMatchers("/api/v1/institutions/**").hasAnyRole("ADMIN", "DIRECTOR")
                .pathMatchers("/api/v1/users/**").hasAnyRole("ADMIN", "DIRECTOR")
                .pathMatchers("/api/v1/students/**").hasAnyRole("ADMIN", "DIRECTOR", "PROFESOR", "AUXILIAR")
                .pathMatchers("/api/v1/attendance/**").hasAnyRole("PROFESOR", "AUXILIAR")
                .pathMatchers("/api/v1/grades/**").hasAnyRole("PROFESOR")

                // Por defecto, autenticado
                .anyExchange().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(jwt -> jwt.jwtAuthenticationConverter(jwtAuthConverter()))
            )
            .build();
    }

    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration config = new CorsConfiguration();
        config.setAllowedOrigins(List.of(
            "http://localhost:3000",
            "http://localhost:5173",
            "https://sigei.vallegrande.edu.pe"
        ));
        config.setAllowedMethods(List.of("GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"));
        config.setAllowedHeaders(List.of("*"));
        config.setAllowCredentials(true);

        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", config);
        return source;
    }
}
```

---

## 9. Seguridad con Keycloak

### 9.1 ¿Por qué Keycloak?

| Característica | Keycloak | Firebase Auth |
|----------------|----------|---------------|
| **On-premise** | ✅ Self-hosted en VPC | ❌ Cloud only |
| **Multi-tenancy** | ✅ Realms + Claims | ⚠️ Limitado |
| **Control total** | ✅ Personalizable | ❌ Limitado |
| **Roles granulares** | ✅ RBAC completo | ⚠️ Básico |
| **Costo** | ✅ Open Source | ⚠️ Pay-per-use |
| **Datos en Perú** | ✅ En tu servidor | ❌ Servidores Google |

### 9.2 Arquitectura de Keycloak

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                                    KEYCLOAK                                              │
│                                 Puerto: 8180                                             │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                         │
│  ┌─────────────────────────────────────────────────────────────────────────────────┐   │
│  │                            REALM: sigei                                          │   │
│  │                                                                                  │   │
│  │  ┌───────────────────────────────────────────────────────────────────────────┐  │   │
│  │  │                           ROLES                                            │  │   │
│  │  ├───────────────────────────────────────────────────────────────────────────┤  │   │
│  │  │  ADMIN      → Administrador de institución                                 │  │   │
│  │  │  DIRECTOR   → Director de institución                                      │  │   │
│  │  │  PROFESOR   → Docente                                                      │  │   │
│  │  │  AUXILIAR   → Personal de apoyo                                            │  │   │
│  │  │  PSICOLOGO  → Psicólogo escolar                                            │  │   │
│  │  │  PADRE      → Padre de familia                                             │  │   │
│  │  │  MADRE      → Madre de familia                                             │  │   │
│  │  │  TUTOR      → Apoderado/Tutor legal                                        │  │   │
│  │  └───────────────────────────────────────────────────────────────────────────┘  │   │
│  │                                                                                  │   │
│  │  ┌───────────────────────────────────────────────────────────────────────────┐  │   │
│  │  │                      CUSTOM CLAIMS (en el JWT)                             │  │   │
│  │  ├───────────────────────────────────────────────────────────────────────────┤  │   │
│  │  │  institution_id  → ID de la institución (tenant)                          │  │   │
│  │  │  classroom_ids   → IDs de aulas asignadas (profesores)                    │  │   │
│  │  │  student_ids     → IDs de hijos (padres)                                  │  │   │
│  │  └───────────────────────────────────────────────────────────────────────────┘  │   │
│  │                                                                                  │   │
│  │  ┌───────────────────────────────────────────────────────────────────────────┐  │   │
│  │  │                          CLIENTS                                           │  │   │
│  │  ├───────────────────────────────────────────────────────────────────────────┤  │   │
│  │  │  sigei-web      → SPA React (public client)                               │  │   │
│  │  │  sigei-gateway  → API Gateway (confidential)                              │  │   │
│  │  └───────────────────────────────────────────────────────────────────────────┘  │   │
│  └─────────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

### 9.3 Estructura del JWT Token

```json
{
  "exp": 1707580800,
  "iat": 1707577200,
  "iss": "http://keycloak:8180/realms/sigei",
  "sub": "user-uuid-12345",

  "institution_id": "inst-uuid-001",
  "institution_name": "I.E.I. Los Jardines",
  "classroom_ids": ["classroom-001", "classroom-002"],
  "student_ids": ["student-001"],

  "realm_access": {
    "roles": ["PROFESOR"]
  },

  "preferred_username": "juan.perez",
  "email": "juan.perez@ejemplo.com",
  "name": "Juan Pérez"
}
```

### 9.4 Matriz de Permisos

```
┌────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                     MATRIZ DE PERMISOS                                                  │
├────────────────┬──────────┬──────────┬──────────┬──────────┬───────────┬───────────────────────────────┤
│ Recurso        │  ADMIN   │ DIRECTOR │ PROFESOR │ AUXILIAR │ PSICOLOGO │  PADRE/MADRE/TUTOR            │
├────────────────┼──────────┼──────────┼──────────┼──────────┼───────────┼───────────────────────────────┤
│ Institutions   │   CRUD   │    R     │    R     │    R     │     R     │     R                         │
│ Users          │   CRUD   │   CRU    │    R     │    R     │     R     │     -                         │
│ Classrooms     │   CRUD   │   CRUD   │    R     │    R     │     R     │     R                         │
│ Students       │   CRUD   │   CRUD   │    RU    │    R     │    RU     │    R (solo sus hijos)         │
│ Enrollments    │   CRUD   │   CRUD   │    R     │    R     │     R     │    R (solo sus hijos)         │
│ Attendance     │   CRUD   │   CRUD   │   CRUD   │   CRU    │     R     │    R (solo sus hijos)         │
│ Grades         │   CRUD   │   CRUD   │   CRUD   │    R     │     R     │    R (solo sus hijos)         │
│ Behavior       │   CRUD   │   CRUD   │   CRU    │   CRU    │    CRU    │    R (solo sus hijos)         │
│ Psychology     │   CRUD   │   CRU    │    R     │    R     │   CRUD    │    R (solo sus hijos)         │
│ Events         │   CRUD   │   CRUD   │    R     │    R     │     R     │     R                         │
│ TeacherAssign  │   CRUD   │   CRUD   │    R     │    -     │     -     │     -                         │
│ Notifications  │   CRUD   │   CRUD   │    RU    │    R     │    RU     │    RU (solo las suyas)        │
├────────────────┴──────────┴──────────┴──────────┴──────────┴───────────┴───────────────────────────────┤
│ Leyenda: C=Create, R=Read, U=Update, D=Delete                                                          │
└────────────────────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 10. Mensajería con RabbitMQ

### 10.1 Arquitectura de RabbitMQ

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                                   RABBITMQ                                               │
│                              Puerto AMQP: 5672                                           │
│                              Puerto Admin: 15672                                         │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                         │
│  ┌─────────────────────────────────────────────────────────────────────────────────┐   │
│  │                           EXCHANGES                                              │   │
│  ├─────────────────────────────────────────────────────────────────────────────────┤   │
│  │                                                                                  │   │
│  │  ┌─────────────────────────┐     ┌─────────────────────────┐                    │   │
│  │  │ sigei.attendance        │     │ sigei.notification      │                    │   │
│  │  │ (Topic Exchange)        │     │ (Direct Exchange)       │                    │   │
│  │  ├─────────────────────────┤     ├─────────────────────────┤                    │   │
│  │  │ Routing Keys:           │     │ Routing Keys:           │                    │   │
│  │  │ • attendance.absent     │     │ • email                 │                    │   │
│  │  │ • attendance.late       │     │ • push                  │                    │   │
│  │  │ • attendance.justified  │     │ • sms                   │                    │   │
│  │  └─────────────────────────┘     │ • whatsapp              │                    │   │
│  │                                   └─────────────────────────┘                    │   │
│  │  ┌─────────────────────────┐     ┌─────────────────────────┐                    │   │
│  │  │ sigei.document          │     │ sigei.dlx               │                    │   │
│  │  │ (Topic Exchange)        │     │ (Dead Letter Exchange)  │                    │   │
│  │  ├─────────────────────────┤     ├─────────────────────────┤                    │   │
│  │  │ • document.shared       │     │ Mensajes fallidos       │                    │   │
│  │  │ • report.generated      │     │ para retry/análisis     │                    │   │
│  │  └─────────────────────────┘     └─────────────────────────┘                    │   │
│  │                                                                                  │   │
│  └─────────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                         │
│  ┌─────────────────────────────────────────────────────────────────────────────────┐   │
│  │                              QUEUES                                              │   │
│  ├─────────────────────────────────────────────────────────────────────────────────┤   │
│  │                                                                                  │   │
│  │  notification.email.queue ─────► ms-notification (Email Worker)                 │   │
│  │  notification.push.queue ──────► ms-notification (Push Worker)                  │   │
│  │  notification.sms.queue ───────► ms-notification (SMS Worker)                   │   │
│  │  notification.whatsapp.queue ──► ms-notification (WhatsApp Worker)              │   │
│  │                                                                                  │   │
│  │  attendance.absent.queue ──────► ms-notification (Alertas de ausencia)          │   │
│  │  document.share.queue ─────────► ms-notification (Envío de documentos)          │   │
│  │                                                                                  │   │
│  └─────────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

### 10.2 Configuración de RabbitMQ

```yaml
# application.yml
spring:
  rabbitmq:
    host: ${RABBITMQ_HOST:rabbitmq}
    port: ${RABBITMQ_PORT:5672}
    username: ${RABBITMQ_USER:sigei}
    password: ${RABBITMQ_PASSWORD:sigei_dev}
    virtual-host: /sigei

    listener:
      simple:
        acknowledge-mode: manual
        prefetch: 10
        retry:
          enabled: true
          initial-interval: 1000
          max-attempts: 3
```

### 10.3 Publisher de Eventos

```java
@Component
@RequiredArgsConstructor
@Slf4j
public class AttendanceEventPublisher {

    private final RabbitTemplate rabbitTemplate;

    public void publishAbsentStudent(AttendanceEvent event) {
        rabbitTemplate.convertAndSend(
            "sigei.attendance",
            "attendance.absent",
            event,
            message -> {
                message.getMessageProperties().setHeader("tenant_id", event.getTenantId());
                return message;
            }
        );
        log.info("Published absent event for student: {}", event.getStudentId());
    }
}
```

### 10.4 Consumer de Eventos

```java
@Component
@RequiredArgsConstructor
@Slf4j
public class AttendanceEventConsumer {

    private final ParentNotificationService notificationService;

    @RabbitListener(queues = "attendance.absent.queue")
    public void handleAbsentStudent(
            AttendanceEvent event,
            Channel channel,
            @Header(AmqpHeaders.DELIVERY_TAG) long tag) throws IOException {

        try {
            log.info("Processing absent notification for student: {}", event.getStudentId());

            // Notificar a los padres
            notificationService.notifyAbsence(event);

            // ACK manual
            channel.basicAck(tag, false);

        } catch (Exception e) {
            log.error("Error processing notification", e);
            // NACK - va al Dead Letter Queue
            channel.basicNack(tag, false, false);
        }
    }
}
```

---

## 11. Microservicio de Notificaciones

### 11.1 Responsabilidades

El `ms-notification` es el único microservicio que envía mensajes a los padres/apoderados:

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                              MS-NOTIFICATION                                             │
│                              Puerto: 9091                                                │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                         │
│  RESPONSABILIDADES:                                                                     │
│  ├── Consumir eventos de RabbitMQ                                                       │
│  ├── Enviar emails (Gmail/SendGrid)                                                     │
│  ├── Enviar push notifications (Firebase Cloud Messaging)                              │
│  ├── Enviar SMS (Twilio)                                                                │
│  ├── Enviar WhatsApp (Twilio WhatsApp API)                                             │
│  ├── Guardar historial de notificaciones                                                │
│  └── Gestionar plantillas de mensajes                                                   │
│                                                                                         │
│  EVENTOS QUE CONSUME:                                                                   │
│  ├── attendance.absent → "Su hijo Juan no asistió hoy"                                 │
│  ├── attendance.late → "Su hijo llegó tarde"                                           │
│  ├── document.shared → "Se ha compartido la libreta de notas"                          │
│  ├── behavior.incident → "Se registró un incidente"                                    │
│  └── grade.published → "Las notas del bimestre están disponibles"                      │
│                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

### 11.2 Flujo de Notificación a Padres

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                    FLUJO: NOTIFICACIÓN DE AUSENCIA A PADRES                              │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                         │
│  ┌────────────┐   ┌────────────┐   ┌────────────┐   ┌────────────┐   ┌────────────┐   │
│  │  Profesor  │   │ Attendance │   │  RabbitMQ  │   │Notification│   │   Padre    │   │
│  │            │   │  Service   │   │            │   │  Service   │   │            │   │
│  └─────┬──────┘   └─────┬──────┘   └─────┬──────┘   └─────┬──────┘   └─────┬──────┘   │
│        │                │                │                │                │          │
│        │ 1. Marca       │                │                │                │          │
│        │    AUSENTE     │                │                │                │          │
│        │───────────────▶│                │                │                │          │
│        │                │                │                │                │          │
│        │                │ 2. Guardar     │                │                │          │
│        │                │    en BD       │                │                │          │
│        │                │────────────────│                │                │          │
│        │                │                │                │                │          │
│        │                │ 3. Publish     │                │                │          │
│        │                │ attendance.    │                │                │          │
│        │                │ absent         │                │                │          │
│        │                │───────────────▶│                │                │          │
│        │                │                │                │                │          │
│        │  4. Response   │                │                │                │          │
│        │◀───────────────│                │                │                │          │
│        │                │                │                │                │          │
│        │                │                │ 5. Consume     │                │          │
│        │                │                │───────────────▶│                │          │
│        │                │                │                │                │          │
│        │                │                │                │ 6. Obtener     │          │
│        │                │                │                │    datos del   │          │
│        │                │                │                │    estudiante  │          │
│        │                │                │                │    y padres    │          │
│        │                │                │                │    (REST)      │          │
│        │                │                │                │────────────────│          │
│        │                │                │                │                │          │
│        │                │                │                │ 7. Enviar      │          │
│        │                │                │                │    WhatsApp    │          │
│        │                │                │                │    + Push      │          │
│        │                │                │                │───────────────▶│          │
│        │                │                │                │                │          │
│        │                │                │                │                │ 📱       │
│        │                │                │                │                │ "Su hijo │
│        │                │                │                │                │ Juan no  │
│        │                │                │                │                │ asistió  │
│        │                │                │                │                │ hoy..."  │
│                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

### 11.3 Tipos de Notificaciones

| Tipo | Canal Principal | Canal Alternativo | Caso de Uso |
|------|-----------------|-------------------|-------------|
| **Ausencia** | WhatsApp | Email | Estudiante no asistió |
| **Tardanza** | Push | WhatsApp | Estudiante llegó tarde |
| **Documento** | Email | Push | Libreta, certificados |
| **Incidente** | WhatsApp + Email | - | Problema de comportamiento |
| **Evento** | Push | Email | Reunión, actividad |
| **Recordatorio** | Push | - | Pago, documentos pendientes |

### 11.4 Estructura del Microservicio

```
ms-notification/
├── src/main/java/pe/edu/vallegrande/notification/
│   ├── domain/
│   │   ├── model/
│   │   │   ├── Notification.java
│   │   │   ├── NotificationChannel.java (EMAIL, PUSH, SMS, WHATSAPP)
│   │   │   ├── NotificationStatus.java (PENDING, SENT, FAILED)
│   │   │   └── NotificationTemplate.java
│   │   └── event/
│   │       ├── AttendanceEvent.java
│   │       ├── DocumentEvent.java
│   │       └── BehaviorEvent.java
│   │
│   ├── application/
│   │   ├── port/
│   │   │   ├── input/
│   │   │   │   └── SendNotificationUseCase.java
│   │   │   └── output/
│   │   │       ├── NotificationPersistencePort.java
│   │   │       ├── EmailSenderPort.java
│   │   │       ├── SmsSenderPort.java
│   │   │       ├── PushSenderPort.java
│   │   │       └── WhatsAppSenderPort.java
│   │   └── service/
│   │       ├── NotificationService.java
│   │       └── ParentNotificationService.java
│   │
│   └── infrastructure/
│       ├── adapter/
│       │   ├── input/
│       │   │   ├── rest/
│       │   │   │   └── NotificationController.java
│       │   │   └── rabbitmq/
│       │   │       ├── AttendanceEventConsumer.java
│       │   │       ├── DocumentEventConsumer.java
│       │   │       └── BehaviorEventConsumer.java
│       │   └── output/
│       │       ├── persistence/
│       │       │   └── NotificationMongoAdapter.java
│       │       ├── email/
│       │       │   └── SendGridEmailAdapter.java
│       │       ├── sms/
│       │       │   └── TwilioSmsAdapter.java
│       │       ├── push/
│       │       │   └── FirebasePushAdapter.java
│       │       └── whatsapp/
│       │           └── TwilioWhatsAppAdapter.java
│       └── config/
│           ├── RabbitMQConfig.java
│           └── SecurityConfig.java
```

---

## 12. Estructura de Carpetas Estandarizada

### 12.1 Estructura del Proyecto Completo

```
sigei-microservices/
├── 📁 infrastructure/
│   ├── 📁 docker/
│   │   ├── docker-compose.yml
│   │   ├── docker-compose.prod.yml
│   │   └── .env.example
│   ├── 📁 nginx/
│   │   ├── nginx.conf
│   │   └── 📁 ssl/
│   ├── 📁 keycloak/
│   │   └── realm-sigei.json
│   ├── 📁 sql/
│   │   └── init.sql
│   └── 📁 scripts/
│       ├── deploy.sh
│       └── backup.sh
│
├── 📁 services/
│   ├── 📁 sigei-gateway/              # API Gateway
│   ├── 📁 ms-institution/             # Instituciones
│   ├── 📁 ms-users/                   # Usuarios
│   ├── 📁 ms-students/                # Estudiantes
│   ├── 📁 ms-academic/                # Académico
│   ├── 📁 ms-enrollments/             # Matrículas
│   ├── 📁 ms-attendance/              # Asistencias
│   ├── 📁 ms-grades/                  # Notas
│   ├── 📁 ms-behavior/                # Comportamiento
│   ├── 📁 ms-psychology/              # Psicología
│   ├── 📁 ms-events/                  # Eventos
│   ├── 📁 ms-teacher-assignment/      # Asignación Docente
│   └── 📁 ms-notification/            # Notificaciones
│
└── 📁 frontend/
    └── 📁 sigei-web/                  # React App
```

### 12.2 Estructura de un Microservicio

```
ms-attendance/
├── 📄 pom.xml
├── 📄 Dockerfile
├── 📄 README.md
│
└── 📁 src/
    ├── 📁 main/
    │   ├── 📁 java/pe/edu/vallegrande/attendance/
    │   │   │
    │   │   ├── 📄 MsAttendanceApplication.java
    │   │   │
    │   │   ├── 📁 domain/                          # 🟢 DOMINIO
    │   │   │   ├── 📁 model/
    │   │   │   │   ├── 📄 Attendance.java          # Entidad
    │   │   │   │   └── 📁 enums/
    │   │   │   │       └── 📄 AttendanceStatus.java
    │   │   │   ├── 📁 event/
    │   │   │   │   └── 📄 AttendanceMarkedEvent.java
    │   │   │   ├── 📁 exception/
    │   │   │   │   └── 📄 AttendanceNotFoundException.java
    │   │   │   └── 📁 repository/
    │   │   │       └── 📄 AttendanceRepository.java  # Interface
    │   │   │
    │   │   ├── 📁 application/                     # 🟡 APLICACIÓN
    │   │   │   ├── 📁 port/
    │   │   │   │   ├── 📁 input/
    │   │   │   │   │   ├── 📄 RegisterAttendanceUseCase.java
    │   │   │   │   │   └── 📄 GetAttendanceUseCase.java
    │   │   │   │   └── 📁 output/
    │   │   │   │       ├── 📄 AttendancePersistencePort.java
    │   │   │   │       └── 📄 AttendanceEventPort.java
    │   │   │   ├── 📁 service/
    │   │   │   │   └── 📄 AttendanceService.java
    │   │   │   └── 📁 dto/
    │   │   │       ├── 📄 AttendanceRequest.java
    │   │   │       └── 📄 AttendanceResponse.java
    │   │   │
    │   │   └── 📁 infrastructure/                  # 🔵 INFRAESTRUCTURA
    │   │       ├── 📁 adapter/
    │   │       │   ├── 📁 input/
    │   │       │   │   └── 📁 rest/
    │   │       │   │       └── 📄 AttendanceController.java
    │   │       │   └── 📁 output/
    │   │       │       ├── 📁 persistence/
    │   │       │       │   ├── 📄 AttendanceR2dbcRepository.java
    │   │       │       │   └── 📄 AttendancePersistenceAdapter.java
    │   │       │       └── 📁 rabbitmq/
    │   │       │           └── 📄 AttendanceRabbitPublisher.java
    │   │       └── 📁 config/
    │   │           ├── 📄 SecurityConfig.java
    │   │           ├── 📄 RabbitMQConfig.java
    │   │           └── 📄 R2dbcConfig.java
    │   │
    │   └── 📁 resources/
    │       ├── 📄 application.yml
    │       └── 📄 application-docker.yml
    │
    └── 📁 test/
        └── 📁 java/pe/edu/vallegrande/attendance/
            ├── 📁 domain/
            ├── 📁 application/
            └── 📁 infrastructure/
```

---

## 13. Configuración por Variables de Entorno

### 13.1 Variables de Entorno Requeridas

```bash
# ============================================================
# BASES DE DATOS
# ============================================================
DATABASE_URL=r2dbc:postgresql://postgres:5432/sigei
DATABASE_USER=sigei
DATABASE_PASSWORD=your_secure_password

MONGODB_URI=mongodb://sigei:your_password@mongodb:27017/sigei?authSource=admin

# ============================================================
# KEYCLOAK
# ============================================================
KEYCLOAK_ISSUER_URI=http://keycloak:8080/realms/sigei
KEYCLOAK_JWK_URI=http://keycloak:8080/realms/sigei/protocol/openid-connect/certs

# ============================================================
# RABBITMQ
# ============================================================
RABBITMQ_HOST=rabbitmq
RABBITMQ_PORT=5672
RABBITMQ_USER=sigei
RABBITMQ_PASSWORD=your_password
RABBITMQ_VHOST=/sigei

# ============================================================
# NOTIFICACIONES - EMAIL
# ============================================================
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=sigei.notificaciones@vallegrande.edu.pe
SMTP_PASSWORD=your_app_password

# ============================================================
# NOTIFICACIONES - SMS/WHATSAPP (Twilio)
# ============================================================
TWILIO_ACCOUNT_SID=your_account_sid
TWILIO_AUTH_TOKEN=your_auth_token
TWILIO_PHONE_NUMBER=+1234567890
TWILIO_WHATSAPP_NUMBER=whatsapp:+14155238886

# ============================================================
# NOTIFICACIONES - PUSH (Firebase)
# ============================================================
FIREBASE_PROJECT_ID=sigei-notifications
FIREBASE_CREDENTIALS=/app/firebase-credentials.json
```

### 13.2 application.yml de un Microservicio

```yaml
server:
  port: 9085

spring:
  application:
    name: ms-attendance

  # PostgreSQL con R2DBC
  r2dbc:
    url: ${DATABASE_URL:r2dbc:postgresql://localhost:5432/sigei}
    username: ${DATABASE_USER:sigei}
    password: ${DATABASE_PASSWORD:sigei_dev}

  # RabbitMQ
  rabbitmq:
    host: ${RABBITMQ_HOST:localhost}
    port: ${RABBITMQ_PORT:5672}
    username: ${RABBITMQ_USER:sigei}
    password: ${RABBITMQ_PASSWORD:sigei_dev}
    virtual-host: ${RABBITMQ_VHOST:/sigei}

  # Seguridad OAuth2
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: ${KEYCLOAK_ISSUER_URI:http://localhost:8180/realms/sigei}

# OpenAPI
springdoc:
  api-docs:
    path: /v3/api-docs
  swagger-ui:
    path: /swagger-ui.html

# Logging
logging:
  level:
    pe.edu.vallegrande: DEBUG
```

---

## 14. Despliegue con Docker Compose

### 14.1 Docker Compose Principal

```yaml
# docker-compose.yml
version: '3.8'

services:
  # ============================================================
  # BASES DE DATOS
  # ============================================================

  postgres:
    image: postgres:16-alpine
    container_name: sigei-postgres
    restart: unless-stopped
    environment:
      POSTGRES_USER: ${DB_USER:-sigei}
      POSTGRES_PASSWORD: ${DB_PASSWORD:-sigei_dev}
      POSTGRES_DB: sigei
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./infrastructure/sql/init.sql:/docker-entrypoint-initdb.d/init.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U sigei"]
      interval: 10s
      timeout: 5s
      retries: 5

  mongodb:
    image: mongo:7.0
    container_name: sigei-mongodb
    restart: unless-stopped
    environment:
      MONGO_INITDB_ROOT_USERNAME: ${MONGO_USER:-sigei}
      MONGO_INITDB_ROOT_PASSWORD: ${MONGO_PASSWORD:-sigei_dev}
    ports:
      - "27017:27017"
    volumes:
      - mongodb_data:/data/db
    healthcheck:
      test: ["CMD", "mongosh", "--eval", "db.adminCommand('ping')"]
      interval: 10s
      timeout: 5s
      retries: 5

  # ============================================================
  # MENSAJERÍA
  # ============================================================

  rabbitmq:
    image: rabbitmq:3.13-management-alpine
    container_name: sigei-rabbitmq
    restart: unless-stopped
    environment:
      RABBITMQ_DEFAULT_USER: ${RABBITMQ_USER:-sigei}
      RABBITMQ_DEFAULT_PASS: ${RABBITMQ_PASSWORD:-sigei_dev}
      RABBITMQ_DEFAULT_VHOST: /sigei
    ports:
      - "5672:5672"
      - "15672:15672"
    volumes:
      - rabbitmq_data:/var/lib/rabbitmq
    healthcheck:
      test: ["CMD", "rabbitmq-diagnostics", "-q", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  # ============================================================
  # SEGURIDAD
  # ============================================================

  keycloak:
    image: quay.io/keycloak/keycloak:24.0.0
    container_name: sigei-keycloak
    restart: unless-stopped
    command: start-dev --import-realm
    environment:
      KEYCLOAK_ADMIN: admin
      KEYCLOAK_ADMIN_PASSWORD: ${KEYCLOAK_PASSWORD:-admin}
      KC_DB: postgres
      KC_DB_URL: jdbc:postgresql://postgres:5432/keycloak
      KC_DB_USERNAME: ${DB_USER:-sigei}
      KC_DB_PASSWORD: ${DB_PASSWORD:-sigei_dev}
    ports:
      - "8180:8080"
    volumes:
      - ./infrastructure/keycloak/realm-sigei.json:/opt/keycloak/data/import/realm-sigei.json
    depends_on:
      postgres:
        condition: service_healthy

  # ============================================================
  # API GATEWAY
  # ============================================================

  gateway:
    build: ./services/sigei-gateway
    container_name: sigei-gateway
    restart: unless-stopped
    ports:
      - "8080:8080"
    environment:
      - SPRING_PROFILES_ACTIVE=docker
      - KEYCLOAK_ISSUER_URI=http://keycloak:8080/realms/sigei
      - KEYCLOAK_JWK_URI=http://keycloak:8080/realms/sigei/protocol/openid-connect/certs
    depends_on:
      - keycloak

  # ============================================================
  # MICROSERVICIOS
  # ============================================================

  ms-institution:
    build: ./services/ms-institution
    container_name: sigei-institution
    restart: unless-stopped
    ports:
      - "9080:9080"
    environment:
      - SPRING_PROFILES_ACTIVE=docker
      - MONGODB_URI=mongodb://${MONGO_USER:-sigei}:${MONGO_PASSWORD:-sigei_dev}@mongodb:27017/sigei?authSource=admin
      - RABBITMQ_HOST=rabbitmq
      - KEYCLOAK_ISSUER_URI=http://keycloak:8080/realms/sigei
    depends_on:
      - mongodb
      - rabbitmq

  ms-users:
    build: ./services/ms-users
    container_name: sigei-users
    restart: unless-stopped
    ports:
      - "9081:9081"
    environment:
      - SPRING_PROFILES_ACTIVE=docker
      - MONGODB_URI=mongodb://${MONGO_USER:-sigei}:${MONGO_PASSWORD:-sigei_dev}@mongodb:27017/sigei?authSource=admin
      - RABBITMQ_HOST=rabbitmq
      - KEYCLOAK_ISSUER_URI=http://keycloak:8080/realms/sigei
    depends_on:
      - mongodb
      - rabbitmq

  ms-students:
    build: ./services/ms-students
    container_name: sigei-students
    restart: unless-stopped
    ports:
      - "9082:9082"
    environment:
      - SPRING_PROFILES_ACTIVE=docker
      - MONGODB_URI=mongodb://${MONGO_USER:-sigei}:${MONGO_PASSWORD:-sigei_dev}@mongodb:27017/sigei?authSource=admin
      - RABBITMQ_HOST=rabbitmq
      - KEYCLOAK_ISSUER_URI=http://keycloak:8080/realms/sigei
    depends_on:
      - mongodb
      - rabbitmq

  ms-academic:
    build: ./services/ms-academic
    container_name: sigei-academic
    restart: unless-stopped
    ports:
      - "9083:9083"
    environment:
      - SPRING_PROFILES_ACTIVE=docker
      - DATABASE_URL=r2dbc:postgresql://postgres:5432/sigei
      - DATABASE_USER=${DB_USER:-sigei}
      - DATABASE_PASSWORD=${DB_PASSWORD:-sigei_dev}
      - RABBITMQ_HOST=rabbitmq
      - KEYCLOAK_ISSUER_URI=http://keycloak:8080/realms/sigei
    depends_on:
      - postgres
      - rabbitmq

  ms-enrollments:
    build: ./services/ms-enrollments
    container_name: sigei-enrollments
    restart: unless-stopped
    ports:
      - "9084:9084"
    environment:
      - SPRING_PROFILES_ACTIVE=docker
      - DATABASE_URL=r2dbc:postgresql://postgres:5432/sigei
      - DATABASE_USER=${DB_USER:-sigei}
      - DATABASE_PASSWORD=${DB_PASSWORD:-sigei_dev}
      - RABBITMQ_HOST=rabbitmq
      - KEYCLOAK_ISSUER_URI=http://keycloak:8080/realms/sigei
    depends_on:
      - postgres
      - rabbitmq

  ms-attendance:
    build: ./services/ms-attendance
    container_name: sigei-attendance
    restart: unless-stopped
    ports:
      - "9085:9085"
    environment:
      - SPRING_PROFILES_ACTIVE=docker
      - DATABASE_URL=r2dbc:postgresql://postgres:5432/sigei
      - DATABASE_USER=${DB_USER:-sigei}
      - DATABASE_PASSWORD=${DB_PASSWORD:-sigei_dev}
      - RABBITMQ_HOST=rabbitmq
      - KEYCLOAK_ISSUER_URI=http://keycloak:8080/realms/sigei
    depends_on:
      - postgres
      - rabbitmq

  ms-grades:
    build: ./services/ms-grades
    container_name: sigei-grades
    restart: unless-stopped
    ports:
      - "9086:9086"
    environment:
      - SPRING_PROFILES_ACTIVE=docker
      - DATABASE_URL=r2dbc:postgresql://postgres:5432/sigei
      - DATABASE_USER=${DB_USER:-sigei}
      - DATABASE_PASSWORD=${DB_PASSWORD:-sigei_dev}
      - RABBITMQ_HOST=rabbitmq
      - KEYCLOAK_ISSUER_URI=http://keycloak:8080/realms/sigei
    depends_on:
      - postgres
      - rabbitmq

  ms-behavior:
    build: ./services/ms-behavior
    container_name: sigei-behavior
    restart: unless-stopped
    ports:
      - "9087:9087"
    environment:
      - SPRING_PROFILES_ACTIVE=docker
      - DATABASE_URL=r2dbc:postgresql://postgres:5432/sigei
      - DATABASE_USER=${DB_USER:-sigei}
      - DATABASE_PASSWORD=${DB_PASSWORD:-sigei_dev}
      - RABBITMQ_HOST=rabbitmq
      - KEYCLOAK_ISSUER_URI=http://keycloak:8080/realms/sigei
    depends_on:
      - postgres
      - rabbitmq

  ms-psychology:
    build: ./services/ms-psychology
    container_name: sigei-psychology
    restart: unless-stopped
    ports:
      - "9088:9088"
    environment:
      - SPRING_PROFILES_ACTIVE=docker
      - DATABASE_URL=r2dbc:postgresql://postgres:5432/sigei
      - DATABASE_USER=${DB_USER:-sigei}
      - DATABASE_PASSWORD=${DB_PASSWORD:-sigei_dev}
      - RABBITMQ_HOST=rabbitmq
      - KEYCLOAK_ISSUER_URI=http://keycloak:8080/realms/sigei
    depends_on:
      - postgres
      - rabbitmq

  ms-events:
    build: ./services/ms-events
    container_name: sigei-events
    restart: unless-stopped
    ports:
      - "9089:9089"
    environment:
      - SPRING_PROFILES_ACTIVE=docker
      - DATABASE_URL=r2dbc:postgresql://postgres:5432/sigei
      - DATABASE_USER=${DB_USER:-sigei}
      - DATABASE_PASSWORD=${DB_PASSWORD:-sigei_dev}
      - RABBITMQ_HOST=rabbitmq
      - KEYCLOAK_ISSUER_URI=http://keycloak:8080/realms/sigei
    depends_on:
      - postgres
      - rabbitmq

  ms-teacher-assignment:
    build: ./services/ms-teacher-assignment
    container_name: sigei-teacher
    restart: unless-stopped
    ports:
      - "9090:9090"
    environment:
      - SPRING_PROFILES_ACTIVE=docker
      - DATABASE_URL=r2dbc:postgresql://postgres:5432/sigei
      - DATABASE_USER=${DB_USER:-sigei}
      - DATABASE_PASSWORD=${DB_PASSWORD:-sigei_dev}
      - RABBITMQ_HOST=rabbitmq
      - KEYCLOAK_ISSUER_URI=http://keycloak:8080/realms/sigei
    depends_on:
      - postgres
      - rabbitmq

  ms-notification:
    build: ./services/ms-notification
    container_name: sigei-notification
    restart: unless-stopped
    ports:
      - "9091:9091"
    environment:
      - SPRING_PROFILES_ACTIVE=docker
      - MONGODB_URI=mongodb://${MONGO_USER:-sigei}:${MONGO_PASSWORD:-sigei_dev}@mongodb:27017/sigei?authSource=admin
      - RABBITMQ_HOST=rabbitmq
      - KEYCLOAK_ISSUER_URI=http://keycloak:8080/realms/sigei
      # Proveedores de notificación
      - SMTP_HOST=${SMTP_HOST:-smtp.gmail.com}
      - SMTP_PORT=${SMTP_PORT:-587}
      - SMTP_USER=${SMTP_USER}
      - SMTP_PASSWORD=${SMTP_PASSWORD}
      - TWILIO_ACCOUNT_SID=${TWILIO_ACCOUNT_SID}
      - TWILIO_AUTH_TOKEN=${TWILIO_AUTH_TOKEN}
      - TWILIO_PHONE_NUMBER=${TWILIO_PHONE_NUMBER}
      - FIREBASE_PROJECT_ID=${FIREBASE_PROJECT_ID}
    depends_on:
      - mongodb
      - rabbitmq

  # ============================================================
  # FRONTEND
  # ============================================================

  frontend:
    build: ./frontend/sigei-web
    container_name: sigei-frontend
    restart: unless-stopped
    ports:
      - "3000:80"
    depends_on:
      - gateway

volumes:
  postgres_data:
  mongodb_data:
  rabbitmq_data:

networks:
  default:
    name: sigei-network
```

### 14.2 Dockerfile Estandarizado

```dockerfile
# Dockerfile para microservicios
FROM eclipse-temurin:17-jdk-alpine as builder

WORKDIR /app

COPY pom.xml .
COPY .mvn .mvn
COPY mvnw .

RUN ./mvnw dependency:go-offline

COPY src src

RUN ./mvnw package -DskipTests

# Imagen final
FROM eclipse-temurin:17-jre-alpine

WORKDIR /app

RUN addgroup -S spring && adduser -S spring -G spring
USER spring:spring

COPY --from=builder /app/target/*.jar app.jar

HEALTHCHECK --interval=30s --timeout=3s --start-period=30s \
    CMD wget -q --spider http://localhost:${SERVER_PORT}/actuator/health || exit 1

ENV JAVA_OPTS="-Xms256m -Xmx512m"

EXPOSE ${SERVER_PORT}

ENTRYPOINT ["sh", "-c", "java ${JAVA_OPTS} -jar app.jar"]
```

### 14.3 Comandos de Despliegue

```bash
# ============================================================
# DESARROLLO LOCAL
# ============================================================

# Iniciar todo
docker-compose up -d

# Ver logs
docker-compose logs -f

# Ver logs de un servicio
docker-compose logs -f ms-notification

# Reconstruir un servicio
docker-compose build ms-attendance
docker-compose up -d ms-attendance

# ============================================================
# PRODUCCIÓN (VPC)
# ============================================================

# Copiar archivos al servidor
scp -r . user@lab.vallegrande.edu.pe:/opt/sigei/

# En el servidor
cd /opt/sigei
docker-compose -f docker-compose.yml -f docker-compose.prod.yml up -d

# Ver estado
docker-compose ps

# Backup de PostgreSQL
docker exec sigei-postgres pg_dump -U sigei sigei > backup_$(date +%Y%m%d).sql

# ============================================================
# URLS DE ACCESO
# ============================================================
# Frontend:     http://localhost:3000
# Gateway API:  http://localhost:8080
# Keycloak:     http://localhost:8180
# RabbitMQ UI:  http://localhost:15672 (sigei/sigei_dev)
# Swagger:      http://localhost:8080/swagger-ui.html
```

---

## 📋 Resumen de Decisiones Arquitectónicas

| Decisión | Elección | Justificación |
|----------|----------|---------------|
| **Arquitectura** | Microservicios + Hexagonal + DDD | Modularidad, mantenibilidad |
| **Comunicación** | REST + RabbitMQ (híbrido) | REST para consultas, RabbitMQ para eventos |
| **Service Discovery** | Docker DNS (nombres) | Docker Compose resuelve por nombre |
| **API Gateway** | Spring Cloud Gateway | Punto único de entrada, centraliza seguridad |
| **Identity Provider** | Keycloak | Multi-tenancy, On-premise, RBAC completo |
| **Base de Datos** | PostgreSQL + MongoDB | Relacional + Documental según dominio |
| **Mensajería** | RabbitMQ | Simple, fácil de debuggear, DLQ nativo |
| **Cache** | ❌ Sin Redis (por ahora) | Simplicidad, se agrega si hay problemas |
| **Config Server** | ❌ Variables de entorno | Docker Compose es suficiente |
| **Multi-Tenancy** | Discriminator Column | Balance costo/aislamiento |
| **Despliegue** | Docker Compose | Adecuado para VPC, sin Kubernetes |

---

*Documento de Arquitectura v2.0 - Febrero 2026*
*Sistema SIGEI - Gestión Educativa Inicial*
*Entorno: VPC lab.vallegrande.edu.pe*
