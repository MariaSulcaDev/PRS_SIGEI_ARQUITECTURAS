# 12 — ARQUITECTURA DE CARPETAS COMPLETA — TODOS LOS MICROSERVICIOS

> **Fecha:** Febrero 2026
> **Sistema:** SIGEI — Sistema Integrado de Gestión Educativa Institucional
> **Contexto:** Colegios PRIVADOS de nivel INICIAL en Perú
> **Patrón:** Arquitectura Hexagonal (Ports & Adapters) + DDD
> **Stack:** Java 17, Spring Boot 3.5.x, WebFlux (Reactivo), PostgreSQL R2DBC, Keycloak

---

## 📋 ÍNDICE DE MICROSERVICIOS

| # | Microservicio | Puerto | Base de Datos | Paquete Base |
|---|--------------|--------|---------------|--------------|
| 1 | [Institution Management](#1-vg-ms-institution-management) | 9080 | PostgreSQL | `pe.edu.vallegrande.sigei.institution` |
| 2 | [Students](#2-vg-ms-students) | 9081 | PostgreSQL | `pe.edu.vallegrande.sigei.students` |
| 3 | [Enrollments](#3-vg-ms-enrollments) | 9082 | PostgreSQL | `pe.edu.vallegrande.sigei.enrollments` |
| 4 | [Users Management](#4-vg-ms-users-management) | 9083 | PostgreSQL | `pe.edu.vallegrande.sigei.users` |
| 5 | [Academic Management](#5-vg-ms-academic-management) | 9084 | PostgreSQL | `pe.edu.vallegrande.sigei.academic` |
| 6 | [Civic Dates](#6-vg-ms-civic-dates) | 9085 | PostgreSQL | `pe.edu.vallegrande.sigei.civicDates` |
| 7 | [Notes](#7-vg-ms-notes) | 9086 | PostgreSQL | `pe.edu.vallegrande.sigei.notes` |
| 8 | [Assistance](#8-vg-ms-assistance) | 9087 | PostgreSQL | `pe.edu.vallegrande.sigei.assistance` |
| 9 | [Disciplinary Management](#9-vg-ms-disciplinary-management) | 9088 | PostgreSQL | `pe.edu.vallegrande.sigei.disciplinary` |
| 10 | [Psychology & Welfare](#10-vg-ms-psychology-welfare) | 9090 | PostgreSQL | `pe.edu.vallegrande.sigei.psychology` |
| 11 | [Teacher Assignment](#11-vg-ms-teacher-assignment) | 9099 | PostgreSQL | `pe.edu.vallegrande.sigei.teacherAssignment` |
| 12 | [Notifications (WhatsApp)](#12-vg-ms-notifications) | 9091 | PostgreSQL | `pe.edu.vallegrande.sigei.notifications` |
| 13 | [API Gateway](#13-vg-ms-gateway) | 8080 | — | `pe.edu.vallegrande.sigei.gateway` |
| 14 | [Eureka Server](#14-vg-ms-eureka-server) | 8761 | — | `pe.edu.vallegrande.sigei.eureka` |

> **IMPORTANTE:** Se unifica el paquete base a `pe.edu.vallegrande.sigei.<modulo>` para TODOS los MS.
> Todos migran a **PostgreSQL + R2DBC** (los 3 que usaban MongoDB: institution, students, users).

---

## 🧩 CONVENCIONES GLOBALES

### Estructura hexagonal estándar (aplica a todos)

```
src/main/java/pe/edu/vallegrande/sigei/<modulo>/
│
├── domain/                           ← CAPA DE DOMINIO (pura, sin frameworks)
│   ├── model/                        ← Entidades y agregados
│   │   ├── XxxEntity.java           ← Entidad raíz (POJO puro, SIN @Table/@Document)
│   │   └── enums/                   ← Enumeraciones del dominio
│   │       └── XxxStatus.java
│   ├── port/                         ← Puertos (interfaces)
│   │   ├── in/                      ← Puertos de ENTRADA (casos de uso)
│   │   │   ├── CreateXxxUseCase.java
│   │   │   ├── FindXxxUseCase.java
│   │   │   └── UpdateXxxUseCase.java
│   │   └── out/                     ← Puertos de SALIDA (repositorios, eventos)
│   │       ├── XxxRepository.java   ← Interfaz pura (NO Spring Data)
│   │       └── EventPublisher.java  ← Interfaz para eventos RabbitMQ
│   ├── exception/                    ← Excepciones de dominio
│   │   ├── XxxNotFoundException.java         extends ResourceNotFoundException
│   │   └── DuplicateXxxException.java        extends BusinessConflictException
│   │
│   └── event/                        ← Eventos de dominio (publicados a RabbitMQ)
│       └── XxxCreatedEvent.java      [record] → se publica al crear/actualizar
│
├── application/                      ← CAPA DE APLICACIÓN (orquestación)
│   ├── service/                     ← Implementación de casos de uso
│   │   └── XxxService.java         ← implements CreateXxxUseCase, FindXxxUseCase...
│   ├── dto/                         ← DTOs de entrada/salida
│   │   ├── request/
│   │   │   ├── CreateXxxRequest.java
│   │   │   └── UpdateXxxRequest.java
│   │   └── response/
│   │       ├── XxxResponse.java
│   │       └── XxxDetailResponse.java
│   └── mapper/                      ← Mappers DTO ↔ Domain
│       └── XxxMapper.java
│
├── infrastructure/                   ← CAPA DE INFRAESTRUCTURA (frameworks/tecnología)
│   ├── adapter/
│   │   ├── in/
│   │   │   └── rest/               ← Adaptadores de ENTRADA (controllers HTTP)
│   │   │       ├── XxxController.java
│   │   │       └── GlobalExceptionHandler.java
│   │   └── out/
│   │       ├── persistence/        ← Adaptadores de SALIDA (base de datos)
│   │       │   ├── entity/
│   │       │   │   └── XxxEntity.java          ← @Table("xxx") — entidad R2DBC
│   │       │   ├── mapper/
│   │       │   │   └── XxxPersistenceMapper.java
│   │       │   ├── repository/
│   │       │   │   └── R2dbcXxxRepository.java ← extends ReactiveCrudRepository
│   │       │   └── XxxPersistenceAdapter.java  ← implements domain XxxRepository
│   │       └── messaging/          ← Adaptadores de SALIDA (RabbitMQ)
│   │           └── RabbitEventPublisher.java   ← implements domain EventPublisher
│   ├── client/                     ← WebClient a otros MS (comunicación síncrona)
│   │   └── XxxClient.java
│   ├── common/                     ← Clases compartidas de infra
│   │   ├── ApiResponse.java
│   │   └── ErrorResponse.java
│   └── config/                     ← Configuración de Spring
│       ├── R2dbcConfig.java
│       ├── SecurityConfig.java
│       ├── RabbitMQConfig.java
│       └── WebClientConfig.java
│
├── shared/                          ← Código compartido dentro del MS
│   └── domain/
│       └── exception/
│           ├── ResourceNotFoundException.java
│           └── BusinessConflictException.java
│
└── XxxApplication.java              ← @SpringBootApplication
│
src/main/resources/
├── application.yml                  ← Configuración base
├── application-dev.yml              ← Perfil desarrollo
├── application-vpc.yml              ← Perfil producción VPC
└── db/migration/                    ← Migraciones Flyway
    ├── V1__create_xxx_table.sql
    └── V2__add_xxx_column.sql
│
src/test/java/pe/edu/vallegrande/sigei/<modulo>/
├── domain/model/                    ← Tests unitarios del dominio
│   └── XxxTest.java
├── application/service/             ← Tests del servicio
│   └── XxxServiceTest.java
└── infrastructure/adapter/in/rest/  ← Tests de integración
    └── XxxControllerTest.java
```

---

## 📂 ARQUITECTURA POR MICROSERVICIO

---

### 1. vg-ms-institution-management

> Gestión de instituciones educativas privadas de nivel inicial y sus aulas.
> **Puerto:** 9080 | **BD:** PostgreSQL schema `institution`

```
src/main/java/pe/edu/vallegrande/sigei/institution/
│
├── domain/
│   ├── model/
│   │   ├── Institution.java
│   │   │   ├── id: String
│   │   │   ├── modularCode: String              ← Código modular UGEL (7 dígitos)
│   │   │   ├── name: String
│   │   │   ├── address: Address                  ← Value Object
│   │   │   ├── contactMethods: List<ContactMethod> ← Value Object
│   │   │   ├── schedules: List<Schedule>         ← Value Object
│   │   │   ├── gradingType: String               ← "CUALITATIVA" (AD/A/B/C)
│   │   │   ├── directorId: String
│   │   │   ├── auxiliaryIds: List<String>
│   │   │   ├── ugel: String
│   │   │   ├── dre: String
│   │   │   ├── status: InstitutionStatus
│   │   │   ├── createdAt: LocalDateTime
│   │   │   └── updatedAt: LocalDateTime
│   │   │
│   │   ├── Classroom.java
│   │   │   ├── id: String
│   │   │   ├── institutionId: String
│   │   │   ├── classroomName: String             ← "Aula Estrellitas"
│   │   │   ├── classroomAge: String              ← "3 años", "4 años", "5 años"
│   │   │   ├── capacity: Integer
│   │   │   ├── color: String
│   │   │   ├── status: ClassroomStatus
│   │   │   ├── createdAt: LocalDateTime
│   │   │   └── updatedAt: LocalDateTime
│   │   │
│   │   ├── Address.java                          ← Value Object (record)
│   │   │   ├── department: String
│   │   │   ├── province: String
│   │   │   ├── district: String
│   │   │   ├── urbanization: String
│   │   │   └── reference: String
│   │   │
│   │   ├── ContactMethod.java                    ← Value Object (record)
│   │   │   ├── type: String                      ← "EMAIL", "PHONE", "WHATSAPP"
│   │   │   └── value: String
│   │   │
│   │   ├── Schedule.java                         ← Value Object (record)
│   │   │   ├── shift: String                     ← "MAÑANA", "TARDE"
│   │   │   ├── startTime: LocalTime
│   │   │   └── endTime: LocalTime
│   │   │
│   │   └── enums/
│   │       ├── InstitutionStatus.java            ← ACTIVE, INACTIVE
│   │       └── ClassroomStatus.java              ← ACTIVE, INACTIVE
│   │
│   ├── port/
│   │   ├── in/
│   │   │   ├── CreateInstitutionUseCase.java
│   │   │   ├── FindInstitutionUseCase.java
│   │   │   ├── UpdateInstitutionUseCase.java
│   │   │   ├── CreateClassroomUseCase.java
│   │   │   ├── FindClassroomUseCase.java
│   │   │   └── UpdateClassroomUseCase.java
│   │   └── out/
│   │       ├── InstitutionRepository.java
│   │       ├── ClassroomRepository.java
│   │       └── InstitutionEventPublisher.java
│   │
│   ├── exception/
│   │   ├── InstitutionNotFoundException.java
│   │   ├── ClassroomNotFoundException.java
│   │   ├── DuplicateModularCodeException.java
│   │   └── ClassroomCapacityException.java
│   │
│   └── event/                        ← Eventos publicados a RabbitMQ
│       ├── InstitutionCreatedEvent.java       [record] institutionId, name, modularCode
│       ├── InstitutionUpdatedEvent.java       [record] institutionId, fieldsChanged
│       ├── ClassroomCreatedEvent.java         [record] classroomId, institutionId, classroomName, ageGroup
│       └── AnnouncementCreatedEvent.java      [record] institutionId, title, message, targetAudience
│
├── application/
│   ├── service/
│   │   ├── InstitutionService.java
│   │   └── ClassroomService.java
│   ├── dto/
│   │   ├── request/
│   │   │   ├── CreateInstitutionRequest.java
│   │   │   ├── UpdateInstitutionRequest.java
│   │   │   ├── CreateClassroomRequest.java
│   │   │   └── UpdateClassroomRequest.java
│   │   └── response/
│   │       ├── InstitutionResponse.java
│   │       ├── InstitutionDetailResponse.java    ← con classrooms incluidos
│   │       └── ClassroomResponse.java
│   └── mapper/
│       ├── InstitutionMapper.java
│       └── ClassroomMapper.java
│
├── infrastructure/
│   ├── adapter/
│   │   ├── in/rest/
│   │   │   ├── InstitutionController.java
│   │   │   │   ├── GET    /api/v1/institutions
│   │   │   │   ├── GET    /api/v1/institutions/active
│   │   │   │   ├── GET    /api/v1/institutions/inactive
│   │   │   │   ├── GET    /api/v1/institutions/{id}
│   │   │   │   ├── GET    /api/v1/institutions/{id}/detail    ← con classrooms y users
│   │   │   │   ├── POST   /api/v1/institutions
│   │   │   │   ├── PUT    /api/v1/institutions/{id}
│   │   │   │   ├── DELETE /api/v1/institutions/{id}
│   │   │   │   └── PATCH  /api/v1/institutions/{id}/restore
│   │   │   │
│   │   │   ├── ClassroomController.java
│   │   │   │   ├── GET    /api/v1/classrooms
│   │   │   │   ├── GET    /api/v1/classrooms/active
│   │   │   │   ├── GET    /api/v1/classrooms/inactive
│   │   │   │   ├── GET    /api/v1/classrooms/{id}
│   │   │   │   ├── GET    /api/v1/classrooms/institution/{institutionId}
│   │   │   │   ├── POST   /api/v1/classrooms
│   │   │   │   ├── PUT    /api/v1/classrooms/{id}
│   │   │   │   ├── DELETE /api/v1/classrooms/{id}
│   │   │   │   └── PATCH  /api/v1/classrooms/{id}/restore
│   │   │   │
│   │   │   └── GlobalExceptionHandler.java
│   │   │
│   │   └── out/
│   │       ├── persistence/
│   │       │   ├── entity/
│   │       │   │   ├── InstitutionEntity.java    ← @Table("institutions")
│   │       │   │   └── ClassroomEntity.java      ← @Table("classrooms")
│   │       │   ├── mapper/
│   │       │   │   ├── InstitutionPersistenceMapper.java
│   │       │   │   └── ClassroomPersistenceMapper.java
│   │       │   ├── repository/
│   │       │   │   ├── R2dbcInstitutionRepository.java
│   │       │   │   └── R2dbcClassroomRepository.java
│   │       │   ├── InstitutionPersistenceAdapter.java
│   │       │   └── ClassroomPersistenceAdapter.java
│   │       └── messaging/
│   │           └── RabbitInstitutionEventPublisher.java
│   │
│   ├── common/
│   │   ├── ApiResponse.java
│   │   └── ErrorResponse.java
│   └── config/
│       ├── R2dbcConfig.java
│       ├── SecurityConfig.java
│       ├── RabbitMQConfig.java
│       └── WebClientConfig.java
│
└── InstitutionApplication.java

src/main/resources/
├── application.yml
├── application-dev.yml
├── application-vpc.yml
└── db/migration/
    ├── V1__create_institutions_table.sql
    ├── V2__create_classrooms_table.sql
    └── V3__create_institution_indexes.sql
```

---

### 2. vg-ms-students

> Gestión de estudiantes de nivel inicial (3-5 años), información personal, de salud y apoderados.
> **Puerto:** 9081 | **BD:** PostgreSQL schema `students`

```
src/main/java/pe/edu/vallegrande/sigei/students/
│
├── domain/
│   ├── model/
│   │   ├── Student.java
│   │   │   ├── id: String
│   │   │   ├── cui: String                       ← Código Único de Identidad
│   │   │   ├── personalInfo: PersonalInfo        ← Value Object
│   │   │   ├── dateOfBirth: LocalDate
│   │   │   ├── address: String
│   │   │   ├── photoUrl: String
│   │   │   ├── institutionId: String
│   │   │   ├── classroomId: String
│   │   │   ├── developmentInfo: DevelopmentInfo  ← Value Object
│   │   │   ├── healthInfo: HealthInfo            ← Value Object
│   │   │   ├── status: StudentStatus
│   │   │   ├── createdAt: LocalDateTime
│   │   │   └── updatedAt: LocalDateTime
│   │   │
│   │   ├── Guardian.java
│   │   │   ├── id: String
│   │   │   ├── studentId: String
│   │   │   ├── firstName: String
│   │   │   ├── lastName: String
│   │   │   ├── relationship: String              ← "PADRE", "MADRE", "OTRO"
│   │   │   ├── documentType: String
│   │   │   ├── documentNumber: String
│   │   │   ├── phone: String
│   │   │   ├── email: String
│   │   │   ├── isEmergencyContact: boolean
│   │   │   └── contactInfo: ContactInfo          ← Value Object
│   │   │
│   │   ├── PersonalInfo.java                     ← Value Object (record)
│   │   │   ├── firstName: String
│   │   │   ├── lastName: String
│   │   │   ├── motherLastName: String
│   │   │   ├── documentType: String
│   │   │   ├── documentNumber: String
│   │   │   └── gender: String
│   │   │
│   │   ├── DevelopmentInfo.java                  ← Value Object (record)
│   │   │   ├── motorDevelopment: String
│   │   │   ├── languageDevelopment: String
│   │   │   ├── socialDevelopment: String
│   │   │   └── observations: String
│   │   │
│   │   ├── HealthInfo.java                       ← Value Object (record)
│   │   │   ├── bloodType: String
│   │   │   ├── allergies: List<String>
│   │   │   ├── medications: List<String>
│   │   │   ├── conditions: List<String>
│   │   │   └── emergencyNotes: String
│   │   │
│   │   ├── ContactInfo.java                      ← Value Object (record)
│   │   │   ├── phone: String
│   │   │   ├── whatsapp: String
│   │   │   └── email: String
│   │   │
│   │   └── enums/
│   │       └── StudentStatus.java                ← ACTIVE, INACTIVE, TRANSFERRED
│   │
│   ├── port/
│   │   ├── in/
│   │   │   ├── CreateStudentUseCase.java
│   │   │   ├── FindStudentUseCase.java
│   │   │   ├── UpdateStudentUseCase.java
│   │   │   └── ManageGuardianUseCase.java
│   │   └── out/
│   │       ├── StudentRepository.java
│   │       ├── GuardianRepository.java
│   │       └── StudentEventPublisher.java
│   │
│   ├── exception/
│   │   ├── StudentNotFoundException.java
│   │   ├── GuardianNotFoundException.java
│   │   └── DuplicateCuiException.java
│   │
│   └── event/                        ← Eventos publicados a RabbitMQ
│       ├── StudentCreatedEvent.java           [record] studentId, institutionId, classroomId, fullName
│       ├── StudentUpdatedEvent.java           [record] studentId, fieldsChanged
│       └── GuardianAddedEvent.java            [record] guardianId, studentId, phone, relationship
│
├── application/
│   ├── service/
│   │   ├── StudentService.java
│   │   └── GuardianService.java
│   ├── dto/
│   │   ├── request/
│   │   │   ├── CreateStudentRequest.java
│   │   │   ├── UpdateStudentRequest.java
│   │   │   ├── CreateGuardianRequest.java
│   │   │   └── UpdateGuardianRequest.java
│   │   └── response/
│   │       ├── StudentResponse.java
│   │       ├── StudentDetailResponse.java        ← con guardians y salud
│   │       └── GuardianResponse.java
│   └── mapper/
│       ├── StudentMapper.java
│       └── GuardianMapper.java
│
├── infrastructure/
│   ├── adapter/
│   │   ├── in/rest/
│   │   │   ├── StudentController.java
│   │   │   │   ├── GET    /api/v1/students
│   │   │   │   ├── GET    /api/v1/students/active
│   │   │   │   ├── GET    /api/v1/students/{id}
│   │   │   │   ├── GET    /api/v1/students/{id}/detail
│   │   │   │   ├── GET    /api/v1/students/cui/{cui}
│   │   │   │   ├── GET    /api/v1/students/classroom/{classroomId}
│   │   │   │   ├── GET    /api/v1/students/institution/{institutionId}
│   │   │   │   ├── POST   /api/v1/students
│   │   │   │   ├── PUT    /api/v1/students/{id}
│   │   │   │   ├── DELETE /api/v1/students/{id}
│   │   │   │   └── PATCH  /api/v1/students/{id}/restore
│   │   │   │
│   │   │   ├── GuardianController.java
│   │   │   │   ├── GET    /api/v1/guardians/student/{studentId}
│   │   │   │   ├── GET    /api/v1/guardians/{id}
│   │   │   │   ├── POST   /api/v1/guardians
│   │   │   │   ├── PUT    /api/v1/guardians/{id}
│   │   │   │   └── DELETE /api/v1/guardians/{id}
│   │   │   │
│   │   │   └── GlobalExceptionHandler.java
│   │   │
│   │   └── out/
│   │       ├── persistence/
│   │       │   ├── entity/
│   │       │   │   ├── StudentEntity.java        ← @Table("students")
│   │       │   │   └── GuardianEntity.java       ← @Table("guardians")
│   │       │   ├── mapper/
│   │       │   │   ├── StudentPersistenceMapper.java
│   │       │   │   └── GuardianPersistenceMapper.java
│   │       │   ├── repository/
│   │       │   │   ├── R2dbcStudentRepository.java
│   │       │   │   └── R2dbcGuardianRepository.java
│   │       │   ├── StudentPersistenceAdapter.java
│   │       │   └── GuardianPersistenceAdapter.java
│   │       └── messaging/
│   │           └── RabbitStudentEventPublisher.java
│   │
│   ├── client/
│   │   ├── InstitutionClient.java                ← WebClient → MS Institution
│   │   └── ClassroomClient.java
│   ├── common/
│   │   ├── ApiResponse.java
│   │   └── ErrorResponse.java
│   └── config/
│       ├── R2dbcConfig.java
│       ├── SecurityConfig.java
│       ├── RabbitMQConfig.java
│       └── WebClientConfig.java
│
└── StudentsApplication.java

src/main/resources/
├── application.yml
├── application-dev.yml
├── application-vpc.yml
└── db/migration/
    ├── V1__create_students_table.sql
    ├── V2__create_guardians_table.sql
    └── V3__create_student_indexes.sql
```

---

### 3. vg-ms-enrollments

> Matrículas escolares, períodos académicos, validación de documentos.
> **Puerto:** 9082 | **BD:** PostgreSQL schema `enrollments`

```
src/main/java/pe/edu/vallegrande/sigei/enrollments/
│
├── domain/
│   ├── model/
│   │   ├── Enrollment.java
│   │   │   ├── id: String
│   │   │   ├── studentId: String
│   │   │   ├── institutionId: String
│   │   │   ├── classroomId: String
│   │   │   ├── academicPeriodId: String
│   │   │   ├── academicYear: String
│   │   │   ├── enrollmentType: EnrollmentType    ← NUEVO, REINGRESO, TRASLADO
│   │   │   ├── enrollmentStatus: EnrollmentStatus ← PENDING, ACTIVE, CANCELLED, COMPLETED
│   │   │   ├── ageGroup: String                  ← "3 años", "4 años", "5 años"
│   │   │   ├── shift: String
│   │   │   ├── section: String
│   │   │   ├── documents: Documents              ← Value Object (checklist docs)
│   │   │   ├── observations: String
│   │   │   ├── registeredByUserId: String
│   │   │   ├── enrollmentDate: LocalDateTime
│   │   │   ├── createdAt: LocalDateTime
│   │   │   └── updatedAt: LocalDateTime
│   │   │
│   │   ├── AcademicPeriod.java
│   │   │   ├── id: String
│   │   │   ├── institutionId: String
│   │   │   ├── academicYear: String
│   │   │   ├── periodName: String
│   │   │   ├── startDate: LocalDate
│   │   │   ├── endDate: LocalDate
│   │   │   ├── enrollmentPeriodStart: LocalDate
│   │   │   ├── enrollmentPeriodEnd: LocalDate
│   │   │   ├── allowLateEnrollment: boolean
│   │   │   ├── lateEnrollmentEndDate: LocalDate
│   │   │   ├── status: PeriodStatus
│   │   │   ├── createdAt: LocalDateTime
│   │   │   └── updatedAt: LocalDateTime
│   │   │
│   │   ├── Documents.java                        ← Value Object (record)
│   │   │   ├── birthCertificate: boolean
│   │   │   ├── studentDni: boolean
│   │   │   ├── guardianDni: boolean
│   │   │   ├── vaccinationCard: boolean
│   │   │   ├── disabilityCertificate: boolean
│   │   │   ├── utilityBill: boolean
│   │   │   ├── psychologicalReport: boolean
│   │   │   └── studentPhoto: boolean
│   │   │
│   │   └── enums/
│   │       ├── EnrollmentStatus.java             ← PENDING, ACTIVE, CANCELLED, COMPLETED
│   │       ├── EnrollmentType.java               ← NUEVO, REINGRESO, TRASLADO
│   │       └── PeriodStatus.java                 ← PLANNING, OPEN, CLOSED
│   │
│   ├── port/
│   │   ├── in/
│   │   │   ├── CreateEnrollmentUseCase.java
│   │   │   ├── FindEnrollmentUseCase.java
│   │   │   ├── UpdateEnrollmentStatusUseCase.java
│   │   │   ├── ValidateEnrollmentUseCase.java
│   │   │   ├── CreateAcademicPeriodUseCase.java
│   │   │   └── FindAcademicPeriodUseCase.java
│   │   └── out/
│   │       ├── EnrollmentRepository.java
│   │       ├── AcademicPeriodRepository.java
│   │       └── EnrollmentEventPublisher.java
│   │
│   ├── exception/
│   │   ├── EnrollmentNotFoundException.java
│   │   ├── AcademicPeriodNotFoundException.java
│   │   ├── DuplicateEnrollmentException.java
│   │   └── EnrollmentPeriodClosedException.java
│   │
│   └── event/                        ← Eventos publicados a RabbitMQ
│       ├── EnrollmentConfirmedEvent.java      [record] enrollmentId, studentId, institutionId, classroomId, academicYear
│       ├── EnrollmentCancelledEvent.java      [record] enrollmentId, studentId, reason
│       └── AcademicPeriodOpenedEvent.java     [record] periodId, institutionId, academicYear, startDate, endDate
│
├── application/
│   ├── service/
│   │   ├── EnrollmentService.java
│   │   └── AcademicPeriodService.java
│   ├── dto/
│   │   ├── request/
│   │   │   ├── CreateEnrollmentRequest.java
│   │   │   ├── UpdateEnrollmentRequest.java
│   │   │   ├── CreateAcademicPeriodRequest.java
│   │   │   └── UpdateStatusRequest.java
│   │   └── response/
│   │       ├── EnrollmentResponse.java
│   │       ├── EnrollmentDetailResponse.java     ← con datos de student e institution
│   │       ├── AcademicPeriodResponse.java
│   │       └── EnrollmentStatisticsResponse.java
│   └── mapper/
│       ├── EnrollmentMapper.java
│       └── AcademicPeriodMapper.java
│
├── infrastructure/
│   ├── adapter/
│   │   ├── in/rest/
│   │   │   ├── EnrollmentController.java
│   │   │   │   ├── POST   /api/v1/enrollments
│   │   │   │   ├── GET    /api/v1/enrollments
│   │   │   │   ├── GET    /api/v1/enrollments/{id}
│   │   │   │   ├── GET    /api/v1/enrollments/institution/{institutionId}
│   │   │   │   ├── GET    /api/v1/enrollments/student/{studentId}
│   │   │   │   ├── GET    /api/v1/enrollments/active
│   │   │   │   ├── GET    /api/v1/enrollments/pending
│   │   │   │   ├── GET    /api/v1/enrollments/statistics/{institutionId}
│   │   │   │   ├── PUT    /api/v1/enrollments/{id}
│   │   │   │   ├── PATCH  /api/v1/enrollments/{id}/status
│   │   │   │   ├── DELETE /api/v1/enrollments/{id}
│   │   │   │   └── PATCH  /api/v1/enrollments/{id}/restore
│   │   │   │
│   │   │   ├── AcademicPeriodController.java
│   │   │   │   ├── POST   /api/v1/academic-periods
│   │   │   │   ├── GET    /api/v1/academic-periods
│   │   │   │   ├── GET    /api/v1/academic-periods/{id}
│   │   │   │   ├── GET    /api/v1/academic-periods/institution/{institutionId}
│   │   │   │   ├── GET    /api/v1/academic-periods/year/{academicYear}
│   │   │   │   ├── PUT    /api/v1/academic-periods/{id}
│   │   │   │   ├── DELETE /api/v1/academic-periods/{id}
│   │   │   │   └── PATCH  /api/v1/academic-periods/{id}/restore
│   │   │   │
│   │   │   └── GlobalExceptionHandler.java
│   │   │
│   │   └── out/
│   │       ├── persistence/
│   │       │   ├── entity/
│   │       │   │   ├── EnrollmentEntity.java     ← @Table("enrollments")
│   │       │   │   └── AcademicPeriodEntity.java ← @Table("academic_periods")
│   │       │   ├── mapper/
│   │       │   │   ├── EnrollmentPersistenceMapper.java
│   │       │   │   └── AcademicPeriodPersistenceMapper.java
│   │       │   ├── repository/
│   │       │   │   ├── R2dbcEnrollmentRepository.java
│   │       │   │   └── R2dbcAcademicPeriodRepository.java
│   │       │   ├── EnrollmentPersistenceAdapter.java
│   │       │   └── AcademicPeriodPersistenceAdapter.java
│   │       └── messaging/
│   │           └── RabbitEnrollmentEventPublisher.java
│   │
│   ├── client/
│   │   ├── InstitutionClient.java
│   │   ├── StudentClient.java
│   │   └── ClassroomClient.java
│   ├── common/
│   │   ├── ApiResponse.java
│   │   └── ErrorResponse.java
│   └── config/
│       ├── R2dbcConfig.java
│       ├── SecurityConfig.java
│       ├── RabbitMQConfig.java
│       └── WebClientConfig.java
│
└── EnrollmentsApplication.java

src/main/resources/
├── application.yml
├── application-dev.yml
├── application-vpc.yml
└── db/migration/
    ├── V1__create_enrollments_table.sql
    ├── V2__create_academic_periods_table.sql
    └── V3__create_enrollment_indexes.sql
```

---

### 4. vg-ms-users-management

> Gestión de usuarios del sistema (directores, docentes, auxiliares, psicólogos, apoderados, secretarias).
> **Puerto:** 9083 | **BD:** PostgreSQL schema `users_management`

```
src/main/java/pe/edu/vallegrande/sigei/users/
│
├── domain/
│   ├── model/
│   │   ├── User.java
│   │   │   ├── id: String
│   │   │   ├── institutionId: String
│   │   │   ├── firstName: String
│   │   │   ├── lastName: String
│   │   │   ├── documentType: String
│   │   │   ├── documentNumber: String
│   │   │   ├── phone: String
│   │   │   ├── address: String
│   │   │   ├── email: String
│   │   │   ├── userName: String
│   │   │   ├── role: UserRole
│   │   │   ├── status: UserStatus
│   │   │   ├── createdAt: LocalDateTime
│   │   │   └── updatedAt: LocalDateTime
│   │   │
│   │   └── enums/
│   │       ├── UserRole.java                     ← DIRECTOR, SUBDIRECTOR, DOCENTE,
│   │       │                                        AUXILIAR, PSICOLOGO, SECRETARIA, APODERADO
│   │       └── UserStatus.java                   ← ACTIVE, INACTIVE
│   │
│   ├── port/
│   │   ├── in/
│   │   │   ├── CreateUserUseCase.java
│   │   │   ├── FindUserUseCase.java
│   │   │   └── UpdateUserUseCase.java
│   │   └── out/
│   │       ├── UserRepository.java
│   │       └── UserEventPublisher.java
│   │
│   ├── exception/
│   │   ├── UserNotFoundException.java
│   │   └── DuplicateDocumentNumberException.java
│   │
│   └── event/                        ← Eventos publicados a RabbitMQ
│       ├── UserCreatedEvent.java              [record] userId, institutionId, role, fullName
│       └── UserDeactivatedEvent.java          [record] userId, institutionId, reason
│
├── application/
│   ├── service/
│   │   └── UserService.java
│   ├── dto/
│   │   ├── request/
│   │   │   ├── CreateUserRequest.java
│   │   │   └── UpdateUserRequest.java
│   │   └── response/
│   │       └── UserResponse.java
│   └── mapper/
│       └── UserMapper.java
│
├── infrastructure/
│   ├── adapter/
│   │   ├── in/rest/
│   │   │   ├── UserController.java
│   │   │   │   ├── GET    /api/v1/users
│   │   │   │   ├── GET    /api/v1/users/{id}
│   │   │   │   ├── GET    /api/v1/users/status/{status}
│   │   │   │   ├── GET    /api/v1/users/role/{role}/status/{status}
│   │   │   │   ├── GET    /api/v1/users/institution/{institutionId}
│   │   │   │   ├── POST   /api/v1/users
│   │   │   │   ├── PUT    /api/v1/users/{id}
│   │   │   │   ├── DELETE /api/v1/users/{id}
│   │   │   │   └── PATCH  /api/v1/users/{id}/restore
│   │   │   │
│   │   │   └── GlobalExceptionHandler.java
│   │   │
│   │   └── out/
│   │       ├── persistence/
│   │       │   ├── entity/
│   │       │   │   └── UserEntity.java           ← @Table("users")
│   │       │   ├── mapper/
│   │       │   │   └── UserPersistenceMapper.java
│   │       │   ├── repository/
│   │       │   │   └── R2dbcUserRepository.java
│   │       │   └── UserPersistenceAdapter.java
│   │       └── messaging/
│   │           └── RabbitUserEventPublisher.java
│   │
│   ├── client/
│   │   └── InstitutionClient.java
│   ├── common/
│   │   ├── ApiResponse.java
│   │   └── ErrorResponse.java
│   └── config/
│       ├── R2dbcConfig.java
│       ├── SecurityConfig.java
│       ├── RabbitMQConfig.java
│       └── WebClientConfig.java
│
└── UsersApplication.java

src/main/resources/
├── application.yml
├── application-dev.yml
├── application-vpc.yml
└── db/migration/
    ├── V1__create_users_table.sql
    └── V2__create_user_indexes.sql
```

---

### 5. vg-ms-academic-management

> Catálogo curricular: cursos, competencias, capacidades y desempeños para nivel inicial.
> **Puerto:** 9084 | **BD:** PostgreSQL schema `academic`

```
src/main/java/pe/edu/vallegrande/sigei/academic/
│
├── domain/
│   ├── model/
│   │   ├── Course.java
│   │   │   ├── id: UUID
│   │   │   ├── institutionId: String
│   │   │   ├── code: String
│   │   │   ├── name: String
│   │   │   ├── areaCurricular: String            ← "Personal Social", "Comunicación", etc.
│   │   │   ├── ageLevel: String                  ← "3 años", "4 años", "5 años"
│   │   │   ├── description: String
│   │   │   ├── status: String
│   │   │   ├── createdAt: LocalDateTime
│   │   │   └── updatedAt: LocalDateTime
│   │   │
│   │   ├── Competency.java
│   │   │   ├── id: UUID
│   │   │   ├── courseId: UUID
│   │   │   ├── institutionId: String
│   │   │   ├── code: String
│   │   │   ├── name: String
│   │   │   ├── description: String
│   │   │   ├── orderIndex: Integer
│   │   │   ├── status: String
│   │   │   ├── createdAt: LocalDateTime
│   │   │   └── updatedAt: LocalDateTime
│   │   │
│   │   ├── Capacity.java
│   │   │   ├── id: UUID
│   │   │   ├── competencyId: UUID
│   │   │   ├── institutionId: String
│   │   │   ├── code: String
│   │   │   ├── name: String
│   │   │   ├── description: String
│   │   │   ├── orderIndex: Integer
│   │   │   ├── status: String
│   │   │   ├── createdAt: LocalDateTime
│   │   │   └── updatedAt: LocalDateTime
│   │   │
│   │   ├── Performance.java
│   │   │   ├── id: UUID
│   │   │   ├── capacityId: UUID
│   │   │   ├── institutionId: String
│   │   │   ├── code: String
│   │   │   ├── description: String
│   │   │   ├── ageLevel: String
│   │   │   ├── orderIndex: Integer
│   │   │   ├── status: String
│   │   │   ├── createdAt: LocalDateTime
│   │   │   └── updatedAt: LocalDateTime
│   │   │
│   │   └── CatalogRegistration.java              ← Agregado para registro masivo
│   │       ├── institutionId: String
│   │       ├── course: Course
│   │       └── competencies: List<CompetencyWithCapacities>
│   │
│   ├── port/
│   │   ├── in/
│   │   │   ├── ManageCourseUseCase.java
│   │   │   ├── ManageCompetencyUseCase.java
│   │   │   ├── ManageCapacityUseCase.java
│   │   │   ├── ManagePerformanceUseCase.java
│   │   │   └── RegisterCatalogUseCase.java
│   │   └── out/
│   │       ├── CourseRepository.java
│   │       ├── CompetencyRepository.java
│   │       ├── CapacityRepository.java
│   │       └── PerformanceRepository.java
│   │
│   ├── exception/
│   │   ├── CourseNotFoundException.java
│   │   ├── CompetencyNotFoundException.java
│   │   └── DuplicateCourseCodeException.java
│   │
│   └── event/                        ← Eventos publicados a RabbitMQ
│       ├── CatalogRegisteredEvent.java        [record] institutionId, courseId, courseName, competencyCount
│       └── CatalogUpdatedEvent.java           [record] institutionId, courseId, changes
│
├── application/
│   ├── service/
│   │   ├── CourseService.java
│   │   ├── CompetencyService.java
│   │   ├── CapacityService.java
│   │   ├── PerformanceService.java
│   │   └── CatalogService.java
│   ├── dto/
│   │   ├── request/
│   │   │   ├── CreateCourseRequest.java
│   │   │   ├── UpdateCourseRequest.java
│   │   │   ├── CreateCompetencyRequest.java
│   │   │   ├── CreateCapacityRequest.java
│   │   │   ├── CreatePerformanceRequest.java
│   │   │   └── CatalogRegistrationRequest.java
│   │   └── response/
│   │       ├── CourseResponse.java
│   │       ├── CompetencyResponse.java
│   │       ├── CapacityResponse.java
│   │       ├── PerformanceResponse.java
│   │       └── CatalogDetailResponse.java
│   └── mapper/
│       ├── CourseMapper.java
│       ├── CompetencyMapper.java
│       ├── CapacityMapper.java
│       └── PerformanceMapper.java
│
├── infrastructure/
│   ├── adapter/
│   │   ├── in/rest/
│   │   │   ├── CourseController.java
│   │   │   │   ├── GET    /api/v1/courses
│   │   │   │   ├── GET    /api/v1/courses/{id}
│   │   │   │   ├── GET    /api/v1/courses/institution/{institutionId}
│   │   │   │   ├── POST   /api/v1/courses
│   │   │   │   ├── PUT    /api/v1/courses/{id}
│   │   │   │   ├── DELETE /api/v1/courses/{id}
│   │   │   │   └── PATCH  /api/v1/courses/{id}/restore
│   │   │   │
│   │   │   ├── CompetencyController.java
│   │   │   │   ├── GET    /api/v1/competencies
│   │   │   │   ├── GET    /api/v1/competencies/{id}
│   │   │   │   ├── GET    /api/v1/competencies/course/{courseId}
│   │   │   │   ├── POST   /api/v1/competencies
│   │   │   │   ├── PUT    /api/v1/competencies/{id}
│   │   │   │   ├── DELETE /api/v1/competencies/{id}
│   │   │   │   └── PATCH  /api/v1/competencies/{id}/restore
│   │   │   │
│   │   │   ├── CapacityController.java
│   │   │   │   ├── GET    /api/v1/capacities
│   │   │   │   ├── GET    /api/v1/capacities/{id}
│   │   │   │   ├── GET    /api/v1/capacities/competency/{competencyId}
│   │   │   │   ├── POST   /api/v1/capacities
│   │   │   │   ├── PUT    /api/v1/capacities/{id}
│   │   │   │   ├── DELETE /api/v1/capacities/{id}
│   │   │   │   └── PATCH  /api/v1/capacities/{id}/restore
│   │   │   │
│   │   │   ├── PerformanceController.java
│   │   │   │   ├── GET    /api/v1/performances
│   │   │   │   ├── GET    /api/v1/performances/{id}
│   │   │   │   ├── GET    /api/v1/performances/capacity/{capacityId}
│   │   │   │   ├── POST   /api/v1/performances
│   │   │   │   ├── PUT    /api/v1/performances/{id}
│   │   │   │   ├── DELETE /api/v1/performances/{id}
│   │   │   │   └── PATCH  /api/v1/performances/{id}/restore
│   │   │   │
│   │   │   ├── CatalogController.java
│   │   │   │   ├── POST   /api/v1/catalog/register
│   │   │   │   ├── PUT    /api/v1/catalog/update
│   │   │   │   ├── GET    /api/v1/catalog/{institutionId}
│   │   │   │   ├── PATCH  /api/v1/catalog/{courseId}/deactivate
│   │   │   │   └── PATCH  /api/v1/catalog/{courseId}/activate
│   │   │   │
│   │   │   └── GlobalExceptionHandler.java
│   │   │
│   │   └── out/persistence/
│   │       ├── entity/
│   │       │   ├── CourseEntity.java             ← @Table("courses")
│   │       │   ├── CompetencyEntity.java         ← @Table("competencies")
│   │       │   ├── CapacityEntity.java           ← @Table("capacities")
│   │       │   └── PerformanceEntity.java        ← @Table("performances")
│   │       ├── mapper/
│   │       │   ├── CoursePersistenceMapper.java
│   │       │   ├── CompetencyPersistenceMapper.java
│   │       │   ├── CapacityPersistenceMapper.java
│   │       │   └── PerformancePersistenceMapper.java
│   │       ├── repository/
│   │       │   ├── R2dbcCourseRepository.java
│   │       │   ├── R2dbcCompetencyRepository.java
│   │       │   ├── R2dbcCapacityRepository.java
│   │       │   └── R2dbcPerformanceRepository.java
│   │       ├── CoursePersistenceAdapter.java
│   │       ├── CompetencyPersistenceAdapter.java
│   │       ├── CapacityPersistenceAdapter.java
│   │       └── PerformancePersistenceAdapter.java
│   │
│   ├── client/
│   │   └── InstitutionClient.java
│   ├── common/
│   │   ├── ApiResponse.java
│   │   └── ErrorResponse.java
│   └── config/
│       ├── R2dbcConfig.java
│       ├── SecurityConfig.java
│       └── WebClientConfig.java
│
└── AcademicApplication.java

src/main/resources/
├── application.yml
├── application-dev.yml
├── application-vpc.yml
└── db/migration/
    ├── V1__create_courses_table.sql
    ├── V2__create_competencies_table.sql
    ├── V3__create_capacities_table.sql
    └── V4__create_performances_table.sql
```

---

### 6. vg-ms-civic-dates

> Calendario cívico escolar, eventos, feriados y calendario académico.
> **Puerto:** 9085 | **BD:** PostgreSQL schema `civic_dates`

```
src/main/java/pe/edu/vallegrande/sigei/civicDates/
│
├── domain/
│   ├── model/
│   │   ├── Event.java
│   │   │   ├── id: Long
│   │   │   ├── institutionId: String
│   │   │   ├── title: String
│   │   │   ├── description: String
│   │   │   ├── startDate: LocalDate
│   │   │   ├── endDate: LocalDate
│   │   │   ├── eventType: String                 ← "CIVICO", "CULTURAL", "RELIGIOSO", "INSTITUCIONAL"
│   │   │   ├── isHoliday: Boolean
│   │   │   ├── isRecurring: Boolean
│   │   │   ├── isNational: Boolean
│   │   │   ├── affectsClasses: Boolean
│   │   │   ├── createdBy: String
│   │   │   ├── status: EventStatus
│   │   │   ├── createdAt: LocalDateTime
│   │   │   └── updatedAt: LocalDateTime
│   │   │
│   │   ├── AcademicCalendar.java
│   │   │   ├── id: Integer
│   │   │   ├── institutionId: String
│   │   │   ├── academicYear: Integer
│   │   │   ├── startDate: LocalDate
│   │   │   ├── endDate: LocalDate
│   │   │   ├── createdAt: LocalDateTime
│   │   │   └── updatedAt: LocalDateTime
│   │   │
│   │   ├── EventCalendar.java                    ← Relación N:N evento-calendario
│   │   │   ├── id: Integer
│   │   │   ├── calendarId: Integer
│   │   │   ├── eventId: Long
│   │   │   └── createdAt: LocalDateTime
│   │   │
│   │   └── enums/
│   │       └── EventStatus.java                  ← ACTIVE, INACTIVE
│   │
│   ├── port/
│   │   ├── in/
│   │   │   ├── ManageEventUseCase.java
│   │   │   └── ManageCalendarUseCase.java
│   │   └── out/
│   │       ├── EventRepository.java
│   │       ├── AcademicCalendarRepository.java
│   │       └── EventCalendarRepository.java
│   │
│   ├── exception/
│   │   ├── EventNotFoundException.java
│   │   └── CalendarNotFoundException.java
│   │
│   └── event/                        ← Eventos publicados a RabbitMQ
│       ├── CivicEventCreatedEvent.java        [record] eventId, institutionId, title, startDate, isHoliday
│       └── EventReminderEvent.java            [record] eventId, institutionId, title, daysUntilEvent
│
├── application/
│   ├── service/
│   │   ├── EventService.java
│   │   └── CalendarService.java
│   ├── dto/
│   │   ├── request/
│   │   │   ├── CreateEventRequest.java
│   │   │   ├── UpdateEventRequest.java
│   │   │   ├── CreateCalendarRequest.java
│   │   │   └── AddEventsToCalendarRequest.java
│   │   └── response/
│   │       ├── EventResponse.java
│   │       ├── CalendarResponse.java
│   │       └── CalendarWithEventsResponse.java
│   └── mapper/
│       ├── EventMapper.java
│       └── CalendarMapper.java
│
├── infrastructure/
│   ├── adapter/
│   │   ├── in/rest/
│   │   │   ├── EventController.java
│   │   │   │   ├── GET    /api/v1/events
│   │   │   │   ├── GET    /api/v1/events/{id}
│   │   │   │   ├── GET    /api/v1/events/institution/{institutionId}
│   │   │   │   ├── GET    /api/v1/events/inactive
│   │   │   │   ├── POST   /api/v1/events
│   │   │   │   ├── PUT    /api/v1/events/{id}
│   │   │   │   ├── DELETE /api/v1/events/{id}
│   │   │   │   └── PATCH  /api/v1/events/{id}/restore
│   │   │   │
│   │   │   ├── CalendarController.java
│   │   │   │   ├── GET    /api/v1/calendars
│   │   │   │   ├── GET    /api/v1/calendars/{id}
│   │   │   │   ├── GET    /api/v1/calendars/institution/{institutionId}
│   │   │   │   ├── GET    /api/v1/calendars/{id}/events
│   │   │   │   ├── POST   /api/v1/calendars
│   │   │   │   ├── POST   /api/v1/calendars/import
│   │   │   │   └── POST   /api/v1/calendars/{id}/events
│   │   │   │
│   │   │   └── GlobalExceptionHandler.java
│   │   │
│   │   └── out/persistence/
│   │       ├── entity/
│   │       │   ├── EventEntity.java              ← @Table("events")
│   │       │   ├── AcademicCalendarEntity.java   ← @Table("academic_calendar")
│   │       │   └── EventCalendarEntity.java      ← @Table("event_calendar")
│   │       ├── mapper/
│   │       │   ├── EventPersistenceMapper.java
│   │       │   └── CalendarPersistenceMapper.java
│   │       ├── repository/
│   │       │   ├── R2dbcEventRepository.java
│   │       │   ├── R2dbcCalendarRepository.java
│   │       │   └── R2dbcEventCalendarRepository.java
│   │       ├── EventPersistenceAdapter.java
│   │       └── CalendarPersistenceAdapter.java
│   │
│   ├── client/
│   │   └── InstitutionClient.java
│   ├── common/
│   │   ├── ApiResponse.java
│   │   └── ErrorResponse.java
│   └── config/
│       ├── R2dbcConfig.java
│       ├── SecurityConfig.java
│       └── WebClientConfig.java
│
└── CivicDatesApplication.java

src/main/resources/
├── application.yml
├── application-dev.yml
├── application-vpc.yml
└── db/migration/
    ├── V1__create_events_table.sql
    ├── V2__create_academic_calendar_table.sql
    └── V3__create_event_calendar_table.sql
```

---

### 7. vg-ms-notes

> Evaluaciones de estudiantes, competencias evaluadas y libretas de notas.
> **Puerto:** 9086 | **BD:** PostgreSQL schema `notes`

```
src/main/java/pe/edu/vallegrande/sigei/notes/
│
├── domain/
│   ├── model/
│   │   ├── StudentEvaluation.java
│   │   │   ├── id: UUID
│   │   │   ├── studentId: String
│   │   │   ├── enrollmentId: String
│   │   │   ├── classroomId: String
│   │   │   ├── institutionId: String
│   │   │   ├── courseId: UUID
│   │   │   ├── competencyId: UUID
│   │   │   ├── academicYear: Integer
│   │   │   ├── achievementLevel: String          ← "AD", "A", "B", "C"
│   │   │   ├── description: String
│   │   │   ├── evaluatedBy: UUID
│   │   │   ├── evaluationDate: LocalDate
│   │   │   ├── observations: String
│   │   │   ├── activityContext: String
│   │   │   ├── evidenceUrls: List<String>
│   │   │   ├── createdAt: LocalDateTime
│   │   │   └── updatedAt: LocalDateTime
│   │   │
│   │   ├── ReportCard.java
│   │   │   ├── id: UUID
│   │   │   ├── studentId: String
│   │   │   ├── enrollmentId: UUID
│   │   │   ├── classroomId: UUID
│   │   │   ├── institutionId: String
│   │   │   ├── academicYear: Integer
│   │   │   ├── academicPeriodId: UUID
│   │   │   ├── periodType: String
│   │   │   ├── periodNumber: Integer
│   │   │   ├── attendancePercentage: BigDecimal
│   │   │   ├── behaviorLevel: String
│   │   │   ├── generalObservations: String
│   │   │   ├── teacherComments: String
│   │   │   ├── overallSummary: String
│   │   │   ├── recommendations: String
│   │   │   ├── status: String                    ← "DRAFT", "APPROVED", "PUBLISHED"
│   │   │   ├── generatedBy: UUID
│   │   │   ├── generatedAt: LocalDateTime
│   │   │   ├── approvedBy: UUID
│   │   │   └── approvedAt: LocalDateTime
│   │   │
│   │   └── enums/
│   │       ├── AchievementLevel.java             ← AD, A, B, C
│   │       └── ReportCardStatus.java             ← DRAFT, APPROVED, PUBLISHED
│   │
│   ├── port/
│   │   ├── in/
│   │   │   ├── ManageEvaluationUseCase.java
│   │   │   ├── FindEvaluationUseCase.java
│   │   │   ├── ManageReportCardUseCase.java
│   │   │   └── FindReportCardUseCase.java
│   │   └── out/
│   │       ├── StudentEvaluationRepository.java
│   │       ├── ReportCardRepository.java
│   │       └── NotesEventPublisher.java
│   │
│   ├── exception/
│   │   ├── EvaluationNotFoundException.java
│   │   ├── ReportCardNotFoundException.java
│   │   └── GradeOutOfRangeException.java
│   │
│   └── event/                        ← Eventos publicados a RabbitMQ
│       ├── EvaluationRegisteredEvent.java     [record] evaluationId, studentId, courseId, achievementLevel
│       └── ReportCardPublishedEvent.java      [record] reportCardId, studentId, institutionId, classroomId, academicYear, periodNumber
│
├── application/
│   ├── service/
│   │   ├── StudentEvaluationService.java
│   │   └── ReportCardService.java
│   ├── dto/
│   │   ├── request/
│   │   │   ├── CreateEvaluationRequest.java
│   │   │   ├── UpdateEvaluationRequest.java
│   │   │   ├── CreateReportCardRequest.java
│   │   │   └── UpdateReportCardRequest.java
│   │   └── response/
│   │       ├── EvaluationResponse.java
│   │       ├── EvaluationDetailResponse.java
│   │       ├── ReportCardResponse.java
│   │       └── ReportCardDetailResponse.java
│   └── mapper/
│       ├── EvaluationMapper.java
│       └── ReportCardMapper.java
│
├── infrastructure/
│   ├── adapter/
│   │   ├── in/rest/
│   │   │   ├── EvaluationController.java
│   │   │   │   ├── POST   /api/v1/evaluations
│   │   │   │   ├── GET    /api/v1/evaluations
│   │   │   │   ├── GET    /api/v1/evaluations/{id}
│   │   │   │   ├── GET    /api/v1/evaluations/student/{studentId}
│   │   │   │   ├── GET    /api/v1/evaluations/classroom/{classroomId}
│   │   │   │   ├── GET    /api/v1/evaluations/course/{courseId}
│   │   │   │   ├── PUT    /api/v1/evaluations/{id}
│   │   │   │   └── DELETE /api/v1/evaluations/{id}
│   │   │   │
│   │   │   ├── ReportCardController.java
│   │   │   │   ├── POST   /api/v1/report-cards
│   │   │   │   ├── GET    /api/v1/report-cards
│   │   │   │   ├── GET    /api/v1/report-cards/{id}
│   │   │   │   ├── GET    /api/v1/report-cards/student/{studentId}
│   │   │   │   ├── PUT    /api/v1/report-cards/{id}
│   │   │   │   ├── PATCH  /api/v1/report-cards/{id}/approve
│   │   │   │   ├── PATCH  /api/v1/report-cards/{id}/publish
│   │   │   │   └── DELETE /api/v1/report-cards/{id}
│   │   │   │
│   │   │   └── GlobalExceptionHandler.java
│   │   │
│   │   └── out/
│   │       ├── persistence/
│   │       │   ├── entity/
│   │       │   │   ├── StudentEvaluationEntity.java ← @Table("student_evaluations")
│   │       │   │   └── ReportCardEntity.java        ← @Table("report_cards")
│   │       │   ├── mapper/
│   │       │   │   ├── EvaluationPersistenceMapper.java
│   │       │   │   └── ReportCardPersistenceMapper.java
│   │       │   ├── repository/
│   │       │   │   ├── R2dbcEvaluationRepository.java
│   │       │   │   └── R2dbcReportCardRepository.java
│   │       │   ├── EvaluationPersistenceAdapter.java
│   │       │   └── ReportCardPersistenceAdapter.java
│   │       └── messaging/
│   │           └── RabbitNotesEventPublisher.java
│   │
│   ├── client/
│   │   ├── InstitutionClient.java
│   │   ├── StudentClient.java
│   │   └── AcademicClient.java                   ← consulta cursos/competencias
│   ├── common/
│   │   ├── ApiResponse.java
│   │   └── ErrorResponse.java
│   └── config/
│       ├── R2dbcConfig.java
│       ├── SecurityConfig.java
│       ├── RabbitMQConfig.java
│       └── WebClientConfig.java
│
└── NotesApplication.java

src/main/resources/
├── application.yml
├── application-dev.yml
├── application-vpc.yml
└── db/migration/
    ├── V1__create_student_evaluations_table.sql
    └── V2__create_report_cards_table.sql
```

---

### 8. vg-ms-assistance

> Control de asistencia diaria, resúmenes mensuales y justificaciones.
> **Puerto:** 9087 | **BD:** PostgreSQL schema `assistance`

```
src/main/java/pe/edu/vallegrande/sigei/assistance/
│
├── domain/
│   ├── model/
│   │   ├── AttendanceRecord.java
│   │   │   ├── id: UUID
│   │   │   ├── studentId: String
│   │   │   ├── classroomId: String
│   │   │   ├── institutionId: String
│   │   │   ├── attendanceDate: LocalDate
│   │   │   ├── academicYear: Integer
│   │   │   ├── attendanceStatus: AttendanceStatus  ← PRESENT, ABSENT, LATE, JUSTIFIED
│   │   │   ├── arrivalTime: LocalTime
│   │   │   ├── departureTime: LocalTime
│   │   │   ├── justified: Boolean
│   │   │   ├── justificationReason: String
│   │   │   ├── justificationDocumentUrl: String
│   │   │   ├── registeredBy: String
│   │   │   ├── registeredAt: LocalDateTime
│   │   │   └── updatedAt: LocalDateTime
│   │   │
│   │   ├── AttendanceSummary.java
│   │   │   ├── id: UUID
│   │   │   ├── studentId: String
│   │   │   ├── classroomId: String
│   │   │   ├── institutionId: String
│   │   │   ├── academicYear: Integer
│   │   │   ├── month: Integer
│   │   │   ├── totalSchoolDays: Integer
│   │   │   ├── daysPresent: Integer
│   │   │   ├── daysAbsent: Integer
│   │   │   ├── daysLate: Integer
│   │   │   ├── daysJustified: Integer
│   │   │   ├── attendancePercentage: BigDecimal
│   │   │   └── lastUpdated: LocalDateTime
│   │   │
│   │   └── enums/
│   │       └── AttendanceStatus.java             ← PRESENT, ABSENT, LATE, JUSTIFIED
│   │
│   ├── port/
│   │   ├── in/
│   │   │   ├── RegisterAttendanceUseCase.java
│   │   │   ├── FindAttendanceUseCase.java
│   │   │   ├── JustifyAttendanceUseCase.java
│   │   │   ├── BulkAttendanceUseCase.java
│   │   │   └── AttendanceSummaryUseCase.java
│   │   └── out/
│   │       ├── AttendanceRecordRepository.java
│   │       ├── AttendanceSummaryRepository.java
│   │       ├── FileStoragePort.java              ← Interfaz para subir justificaciones
│   │       └── AttendanceEventPublisher.java
│   │
│   ├── exception/
│   │   ├── AttendanceNotFoundException.java
│   │   └── AttendanceAlreadyRegisteredException.java
│   │
│   └── event/                        ← Eventos publicados a RabbitMQ
│       ├── AttendanceAbsentEvent.java         [record] studentId, institutionId, classroomId, date, registeredBy
│       ├── AttendanceLateEvent.java           [record] studentId, institutionId, classroomId, date, arrivalTime
│       └── AttendanceDailySummaryEvent.java   [record] institutionId, classroomId, date, presentCount, absentCount, lateCount
│
├── application/
│   ├── service/
│   │   ├── AttendanceService.java
│   │   ├── AttendanceSummaryService.java
│   │   └── FileStorageService.java
│   ├── dto/
│   │   ├── request/
│   │   │   ├── CreateAttendanceRequest.java
│   │   │   ├── BulkAttendanceRequest.java
│   │   │   ├── JustifyAttendanceRequest.java
│   │   │   └── RecalculateSummaryRequest.java
│   │   └── response/
│   │       ├── AttendanceResponse.java
│   │       ├── AttendanceSummaryResponse.java
│   │       └── AttendanceStatisticsResponse.java
│   └── mapper/
│       ├── AttendanceMapper.java
│       └── AttendanceSummaryMapper.java
│
├── infrastructure/
│   ├── adapter/
│   │   ├── in/rest/
│   │   │   ├── AttendanceController.java
│   │   │   │   ├── POST   /api/v1/attendance
│   │   │   │   ├── POST   /api/v1/attendance/bulk
│   │   │   │   ├── GET    /api/v1/attendance/{id}
│   │   │   │   ├── GET    /api/v1/attendance/student/{studentId}
│   │   │   │   ├── GET    /api/v1/attendance/classroom/{classroomId}
│   │   │   │   ├── GET    /api/v1/attendance/classroom/{classroomId}/date/{date}
│   │   │   │   ├── GET    /api/v1/attendance/institution/{institutionId}
│   │   │   │   ├── GET    /api/v1/attendance/student/{studentId}/stats
│   │   │   │   ├── PUT    /api/v1/attendance/{id}
│   │   │   │   ├── PATCH  /api/v1/attendance/{id}/justify
│   │   │   │   └── DELETE /api/v1/attendance/{id}
│   │   │   │
│   │   │   ├── AttendanceSummaryController.java
│   │   │   │   ├── GET    /api/v1/attendance-summary/student/{studentId}
│   │   │   │   ├── GET    /api/v1/attendance-summary/classroom/{classroomId}
│   │   │   │   ├── GET    /api/v1/attendance-summary/institution/{institutionId}
│   │   │   │   ├── GET    /api/v1/attendance-summary/statistics
│   │   │   │   └── POST   /api/v1/attendance-summary/recalculate
│   │   │   │
│   │   │   ├── FileUploadController.java
│   │   │   │   ├── POST   /api/v1/files/upload
│   │   │   │   ├── DELETE /api/v1/files/{fileId}
│   │   │   │   └── GET    /api/v1/files/list
│   │   │   │
│   │   │   └── GlobalExceptionHandler.java
│   │   │
│   │   └── out/
│   │       ├── persistence/
│   │       │   ├── entity/
│   │       │   │   ├── AttendanceRecordEntity.java   ← @Table("attendance_records")
│   │       │   │   └── AttendanceSummaryEntity.java  ← @Table("attendance_summary")
│   │       │   ├── mapper/
│   │       │   │   ├── AttendancePersistenceMapper.java
│   │       │   │   └── SummaryPersistenceMapper.java
│   │       │   ├── repository/
│   │       │   │   ├── R2dbcAttendanceRepository.java
│   │       │   │   └── R2dbcSummaryRepository.java
│   │       │   ├── AttendancePersistenceAdapter.java
│   │       │   └── SummaryPersistenceAdapter.java
│   │       ├── storage/
│   │       │   └── SupabaseStorageAdapter.java   ← implements FileStoragePort
│   │       └── messaging/
│   │           └── RabbitAttendanceEventPublisher.java
│   │
│   ├── client/
│   │   ├── InstitutionClient.java
│   │   ├── StudentClient.java
│   │   └── ClassroomClient.java
│   ├── common/
│   │   ├── ApiResponse.java
│   │   └── ErrorResponse.java
│   └── config/
│       ├── R2dbcConfig.java
│       ├── SecurityConfig.java
│       ├── RabbitMQConfig.java
│       ├── SupabaseConfig.java
│       └── WebClientConfig.java
│
└── AssistanceApplication.java

src/main/resources/
├── application.yml
├── application-dev.yml
├── application-vpc.yml
└── db/migration/
    ├── V1__create_attendance_records_table.sql
    ├── V2__create_attendance_summary_table.sql
    └── V3__create_attendance_indexes.sql
```

---

### 9. vg-ms-disciplinary-management

> Registro de comportamiento, incidentes y seguimiento disciplinario.
> **Puerto:** 9088 | **BD:** PostgreSQL schema `disciplinary`

```
src/main/java/pe/edu/vallegrande/sigei/disciplinary/
│
├── domain/
│   ├── model/
│   │   ├── BehaviorRecord.java
│   │   │   ├── id: UUID
│   │   │   ├── studentId: String
│   │   │   ├── classroomId: String
│   │   │   ├── institutionId: String
│   │   │   ├── recordDate: LocalDate
│   │   │   ├── academicYear: Integer
│   │   │   ├── behaviorType: BehaviorType        ← POSITIVE, NEGATIVE
│   │   │   ├── behaviorLevel: BehaviorLevel      ← MINOR, MODERATE, SEVERE
│   │   │   ├── description: String
│   │   │   ├── context: String
│   │   │   ├── actionTaken: String
│   │   │   ├── requiresFollowUp: Boolean
│   │   │   ├── recordedBy: String
│   │   │   ├── recordedAt: LocalDateTime
│   │   │   └── updatedAt: LocalDateTime
│   │   │
│   │   ├── Incident.java
│   │   │   ├── id: UUID
│   │   │   ├── studentId: String
│   │   │   ├── classroomId: String
│   │   │   ├── institutionId: String
│   │   │   ├── incidentDate: LocalDate
│   │   │   ├── incidentTime: LocalTime
│   │   │   ├── academicYear: Integer
│   │   │   ├── incidentType: IncidentType        ← PHYSICAL, VERBAL, etc.
│   │   │   ├── severityLevel: SeverityLevel      ← LOW, MEDIUM, HIGH, CRITICAL
│   │   │   ├── description: String
│   │   │   ├── location: String
│   │   │   ├── witnesses: String
│   │   │   ├── otherStudentsInvolved: List<String>
│   │   │   ├── immediateAction: String
│   │   │   ├── parentsNotified: Boolean
│   │   │   ├── notificationDate: LocalDate
│   │   │   ├── followUpRequired: Boolean
│   │   │   ├── status: IncidentStatus            ← OPEN, IN_PROGRESS, RESOLVED, CLOSED
│   │   │   ├── reportedBy: String
│   │   │   ├── reportedAt: LocalDateTime
│   │   │   ├── resolvedBy: String
│   │   │   └── resolvedAt: LocalDateTime
│   │   │
│   │   └── enums/
│   │       ├── BehaviorType.java
│   │       ├── BehaviorLevel.java
│   │       ├── IncidentType.java
│   │       ├── SeverityLevel.java
│   │       └── IncidentStatus.java
│   │
│   ├── port/
│   │   ├── in/
│   │   │   ├── ManageBehaviorRecordUseCase.java
│   │   │   ├── FindBehaviorRecordUseCase.java
│   │   │   ├── ManageIncidentUseCase.java
│   │   │   └── FindIncidentUseCase.java
│   │   └── out/
│   │       ├── BehaviorRecordRepository.java
│   │       ├── IncidentRepository.java
│   │       └── DisciplinaryEventPublisher.java
│   │
│   ├── exception/
│   │   ├── BehaviorRecordNotFoundException.java
│   │   └── IncidentNotFoundException.java
│   │
│   └── event/                        ← Eventos publicados a RabbitMQ
│       ├── IncidentCreatedEvent.java          [record] incidentId, studentId, institutionId, incidentType, severityLevel, description
│       ├── IncidentResolvedEvent.java         [record] incidentId, studentId, resolvedBy, resolution
│       └── BehaviorAlertEvent.java            [record] studentId, institutionId, behaviorType, behaviorLevel, description
│
├── application/
│   ├── service/
│   │   ├── BehaviorRecordService.java
│   │   └── IncidentService.java
│   ├── dto/
│   │   ├── request/
│   │   │   ├── CreateBehaviorRecordRequest.java
│   │   │   ├── UpdateBehaviorRecordRequest.java
│   │   │   ├── CreateIncidentRequest.java
│   │   │   └── UpdateIncidentRequest.java
│   │   └── response/
│   │       ├── BehaviorRecordResponse.java
│   │       ├── BehaviorRecordDetailResponse.java ← con datos de student enriquecidos
│   │       ├── IncidentResponse.java
│   │       └── IncidentDetailResponse.java
│   └── mapper/
│       ├── BehaviorRecordMapper.java
│       └── IncidentMapper.java
│
├── infrastructure/
│   ├── adapter/
│   │   ├── in/rest/
│   │   │   ├── BehaviorRecordController.java
│   │   │   │   ├── POST   /api/v1/behavior-records
│   │   │   │   ├── GET    /api/v1/behavior-records
│   │   │   │   ├── GET    /api/v1/behavior-records/student/{studentId}
│   │   │   │   ├── GET    /api/v1/behavior-records/classroom/{classroomId}
│   │   │   │   ├── PUT    /api/v1/behavior-records/{id}
│   │   │   │   └── DELETE /api/v1/behavior-records/{id}
│   │   │   │
│   │   │   ├── IncidentController.java
│   │   │   │   ├── POST   /api/v1/incidents
│   │   │   │   ├── GET    /api/v1/incidents
│   │   │   │   ├── GET    /api/v1/incidents/{id}
│   │   │   │   ├── GET    /api/v1/incidents/student/{studentId}
│   │   │   │   ├── GET    /api/v1/incidents/status/{status}
│   │   │   │   ├── PUT    /api/v1/incidents/{id}
│   │   │   │   ├── PATCH  /api/v1/incidents/{id}/resolve
│   │   │   │   └── DELETE /api/v1/incidents/{id}
│   │   │   │
│   │   │   └── GlobalExceptionHandler.java
│   │   │
│   │   └── out/
│   │       ├── persistence/
│   │       │   ├── entity/
│   │       │   │   ├── BehaviorRecordEntity.java ← @Table("behavior_records")
│   │       │   │   └── IncidentEntity.java       ← @Table("incidents")
│   │       │   ├── mapper/
│   │       │   │   ├── BehaviorPersistenceMapper.java
│   │       │   │   └── IncidentPersistenceMapper.java
│   │       │   ├── repository/
│   │       │   │   ├── R2dbcBehaviorRepository.java
│   │       │   │   └── R2dbcIncidentRepository.java
│   │       │   ├── BehaviorPersistenceAdapter.java
│   │       │   └── IncidentPersistenceAdapter.java
│   │       └── messaging/
│   │           └── RabbitDisciplinaryEventPublisher.java
│   │
│   ├── client/
│   │   ├── InstitutionClient.java
│   │   ├── StudentClient.java
│   │   ├── ClassroomClient.java
│   │   └── UserClient.java
│   ├── common/
│   │   ├── ApiResponse.java
│   │   └── ErrorResponse.java
│   └── config/
│       ├── R2dbcConfig.java
│       ├── SecurityConfig.java
│       ├── RabbitMQConfig.java
│       └── WebClientConfig.java
│
└── DisciplinaryApplication.java

src/main/resources/
├── application.yml
├── application-dev.yml
├── application-vpc.yml
└── db/migration/
    ├── V1__create_behavior_records_table.sql
    ├── V2__create_incidents_table.sql
    └── V3__create_disciplinary_indexes.sql
```

---

### 10. vg-ms-psychology-welfare

> Evaluaciones psicológicas, apoyo a necesidades especiales y bienestar estudiantil.
> **Puerto:** 9090 | **BD:** PostgreSQL schema `psychology`

```
src/main/java/pe/edu/vallegrande/sigei/psychology/
│
├── domain/
│   ├── model/
│   │   ├── PsychologicalEvaluation.java
│   │   │   ├── id: UUID
│   │   │   ├── studentId: UUID
│   │   │   ├── classroomId: UUID
│   │   │   ├── institutionId: UUID
│   │   │   ├── evaluationDate: LocalDate
│   │   │   ├── academicYear: Integer
│   │   │   ├── evaluationType: EvaluationType    ← INITIAL, FOLLOW_UP, FINAL, EMERGENCY
│   │   │   ├── evaluationReason: String
│   │   │   ├── emotionalDevelopment: DevelopmentLevel
│   │   │   ├── socialDevelopment: DevelopmentLevel
│   │   │   ├── cognitiveDevelopment: DevelopmentLevel
│   │   │   ├── motorDevelopment: DevelopmentLevel
│   │   │   ├── observations: String
│   │   │   ├── recommendations: String
│   │   │   ├── requiresFollowUp: Boolean
│   │   │   ├── followUpFrequency: String
│   │   │   ├── evaluatedBy: UUID
│   │   │   ├── status: Status
│   │   │   ├── evaluatedAt: LocalDateTime
│   │   │   └── updatedAt: LocalDateTime
│   │   │
│   │   ├── SpecialNeedsSupport.java
│   │   │   ├── id: UUID
│   │   │   ├── studentId: UUID
│   │   │   ├── classroomId: UUID
│   │   │   ├── institutionId: UUID
│   │   │   ├── academicYear: Integer
│   │   │   ├── diagnosis: String
│   │   │   ├── diagnosisDate: LocalDate
│   │   │   ├── diagnosedBy: String
│   │   │   ├── supportType: SupportType          ← SPEECH_THERAPY, OCCUPATIONAL_THERAPY, etc.
│   │   │   ├── description: String
│   │   │   ├── adaptationsRequired: List<String>
│   │   │   ├── supportMaterials: List<String>
│   │   │   ├── specialistInvolved: String
│   │   │   ├── progressNotes: String
│   │   │   ├── lastReviewDate: LocalDate
│   │   │   ├── nextReviewDate: LocalDate
│   │   │   ├── status: Status
│   │   │   ├── createdAt: LocalDateTime
│   │   │   └── updatedAt: LocalDateTime
│   │   │
│   │   └── enums/
│   │       ├── EvaluationType.java               ← INITIAL, FOLLOW_UP, FINAL, EMERGENCY
│   │       ├── DevelopmentLevel.java             ← ADVANCED, EXPECTED, IN_PROGRESS, NEEDS_SUPPORT
│   │       ├── SupportType.java                  ← SPEECH_THERAPY, OCCUPATIONAL, BEHAVIORAL, etc.
│   │       └── Status.java                       ← ACTIVE, INACTIVE
│   │
│   ├── port/
│   │   ├── in/
│   │   │   ├── ManageEvaluationUseCase.java
│   │   │   ├── FindEvaluationUseCase.java
│   │   │   ├── ManageSpecialNeedsUseCase.java
│   │   │   └── FindSpecialNeedsUseCase.java
│   │   └── out/
│   │       ├── PsychologicalEvaluationRepository.java
│   │       ├── SpecialNeedsSupportRepository.java
│   │       └── PsychologyEventPublisher.java
│   │
│   ├── exception/
│   │   ├── EvaluationNotFoundException.java
│   │   └── SpecialNeedsSupportNotFoundException.java
│   │
│   └── event/                        ← Eventos publicados a RabbitMQ
│       ├── PsychologicalEvaluationCompletedEvent.java [record] evaluationId, studentId, institutionId, evaluationType, requiresFollowUp
│       └── FollowUpDueEvent.java              [record] evaluationId, studentId, institutionId, dueDate, followUpFrequency
│
├── application/
│   ├── service/
│   │   ├── PsychologicalEvaluationService.java
│   │   └── SpecialNeedsSupportService.java
│   ├── dto/
│   │   ├── request/
│   │   │   ├── CreateEvaluationRequest.java
│   │   │   ├── UpdateEvaluationRequest.java
│   │   │   ├── CreateSpecialNeedsRequest.java
│   │   │   └── UpdateSpecialNeedsRequest.java
│   │   └── response/
│   │       ├── EvaluationResponse.java
│   │       ├── EvaluationDetailResponse.java
│   │       ├── SpecialNeedsResponse.java
│   │       └── ReferenceDataResponse.java
│   └── mapper/
│       ├── EvaluationMapper.java
│       └── SpecialNeedsMapper.java
│
├── infrastructure/
│   ├── adapter/
│   │   ├── in/rest/
│   │   │   ├── PsychologicalEvaluationController.java
│   │   │   │   ├── POST   /api/v1/psychological-evaluations
│   │   │   │   ├── GET    /api/v1/psychological-evaluations
│   │   │   │   ├── GET    /api/v1/psychological-evaluations/{id}
│   │   │   │   ├── GET    /api/v1/psychological-evaluations/student/{studentId}
│   │   │   │   ├── GET    /api/v1/psychological-evaluations/classroom/{classroomId}
│   │   │   │   ├── GET    /api/v1/psychological-evaluations/institution/{institutionId}
│   │   │   │   ├── GET    /api/v1/psychological-evaluations/active
│   │   │   │   ├── GET    /api/v1/psychological-evaluations/inactive
│   │   │   │   ├── PUT    /api/v1/psychological-evaluations/{id}
│   │   │   │   ├── PATCH  /api/v1/psychological-evaluations/{id}/deactivate
│   │   │   │   └── PATCH  /api/v1/psychological-evaluations/{id}/reactivate
│   │   │   │
│   │   │   ├── SpecialNeedsSupportController.java
│   │   │   │   ├── POST   /api/v1/special-needs-support
│   │   │   │   ├── GET    /api/v1/special-needs-support
│   │   │   │   ├── GET    /api/v1/special-needs-support/{id}
│   │   │   │   ├── GET    /api/v1/special-needs-support/student/{studentId}
│   │   │   │   ├── GET    /api/v1/special-needs-support/type/{supportType}
│   │   │   │   ├── PUT    /api/v1/special-needs-support/{id}
│   │   │   │   ├── DELETE /api/v1/special-needs-support/{id}
│   │   │   │   └── PATCH  /api/v1/special-needs-support/{id}/activate
│   │   │   │
│   │   │   ├── ReferenceDataController.java
│   │   │   │   ├── GET    /api/v1/reference-data/students
│   │   │   │   ├── GET    /api/v1/reference-data/classrooms
│   │   │   │   ├── GET    /api/v1/reference-data/institutions
│   │   │   │   └── GET    /api/v1/reference-data/evaluators
│   │   │   │
│   │   │   └── GlobalExceptionHandler.java
│   │   │
│   │   └── out/
│   │       ├── persistence/
│   │       │   ├── entity/
│   │       │   │   ├── EvaluationEntity.java         ← @Table("psychological_evaluations")
│   │       │   │   └── SpecialNeedsSupportEntity.java ← @Table("special_needs_support")
│   │       │   ├── mapper/
│   │       │   │   ├── EvaluationPersistenceMapper.java
│   │       │   │   └── SpecialNeedsPersistenceMapper.java
│   │       │   ├── repository/
│   │       │   │   ├── R2dbcEvaluationRepository.java
│   │       │   │   └── R2dbcSpecialNeedsRepository.java
│   │       │   ├── EvaluationPersistenceAdapter.java
│   │       │   └── SpecialNeedsPersistenceAdapter.java
│   │       └── messaging/
│   │           └── RabbitPsychologyEventPublisher.java
│   │
│   ├── client/
│   │   ├── InstitutionClient.java
│   │   ├── StudentClient.java
│   │   ├── ClassroomClient.java
│   │   └── UserClient.java
│   ├── common/
│   │   ├── ApiResponse.java
│   │   └── ErrorResponse.java
│   └── config/
│       ├── R2dbcConfig.java
│       ├── SecurityConfig.java
│       ├── RabbitMQConfig.java
│       └── WebClientConfig.java
│
└── PsychologyApplication.java

src/main/resources/
├── application.yml
├── application-dev.yml
├── application-vpc.yml
└── db/migration/
    ├── V1__create_psychological_evaluations_table.sql
    ├── V2__create_special_needs_support_table.sql
    └── V3__create_psychology_indexes.sql
```

---

### 11. vg-ms-teacher-assignment

> Asignación de docentes a aulas, horarios y gestión de carga académica.
> **Puerto:** 9099 | **BD:** PostgreSQL schema `teacher_assignment`

```
src/main/java/pe/edu/vallegrande/sigei/teacherAssignment/
│
├── domain/
│   ├── model/
│   │   ├── TeacherAssignment.java
│   │   │   ├── id: UUID
│   │   │   ├── teacherUserId: String
│   │   │   ├── institutionId: String
│   │   │   ├── assignmentType: AssignmentType    ← REGULAR, SUBSTITUTE, SUPPORT
│   │   │   ├── status: Status                    ← ACTIVE, INACTIVE, DELETED
│   │   │   ├── startDate: LocalDate
│   │   │   ├── endDate: LocalDate
│   │   │   ├── academicYear: String
│   │   │   ├── notes: String
│   │   │   ├── createdAt: LocalDateTime
│   │   │   ├── updatedAt: LocalDateTime
│   │   │   └── deletedAt: LocalDateTime
│   │   │
│   │   ├── TeacherAssignmentClassroom.java
│   │   │   ├── id: UUID
│   │   │   ├── assignmentId: UUID
│   │   │   ├── classroomId: String
│   │   │   ├── isPrimary: boolean
│   │   │   ├── createdAt: LocalDateTime
│   │   │   └── updatedAt: LocalDateTime
│   │   │
│   │   ├── TeacherAssignmentSchedule.java
│   │   │   ├── id: UUID
│   │   │   ├── assignmentId: UUID
│   │   │   ├── classroomId: String
│   │   │   ├── dayOfWeek: DayOfWeek
│   │   │   ├── startTime: LocalTime
│   │   │   ├── endTime: LocalTime
│   │   │   ├── sessionType: SessionType          ← REGULAR, TUTORIAL, EXTRA
│   │   │   └── createdAt: LocalDateTime
│   │   │
│   │   └── enums/
│   │       ├── AssignmentType.java               ← REGULAR, SUBSTITUTE, SUPPORT
│   │       ├── Status.java                       ← ACTIVE, INACTIVE, DELETED
│   │       ├── DayOfWeek.java                    ← MONDAY..FRIDAY
│   │       └── SessionType.java                  ← REGULAR, TUTORIAL, EXTRA
│   │
│   ├── port/
│   │   ├── in/
│   │   │   ├── ManageAssignmentUseCase.java
│   │   │   ├── FindAssignmentUseCase.java
│   │   │   ├── ManageClassroomAssignmentUseCase.java
│   │   │   └── ManageScheduleUseCase.java
│   │   └── out/
│   │       ├── TeacherAssignmentRepository.java
│   │       ├── AssignmentClassroomRepository.java
│   │       ├── AssignmentScheduleRepository.java
│   │       └── AssignmentEventPublisher.java
│   │
│   ├── exception/
│   │   ├── AssignmentNotFoundException.java
│   │   └── AssignmentConflictException.java
│   │
│   └── event/                        ← Eventos publicados a RabbitMQ
│       ├── AssignmentCreatedEvent.java        [record] assignmentId, teacherUserId, institutionId, classroomIds, academicYear
│       └── AssignmentUpdatedEvent.java        [record] assignmentId, teacherUserId, changes
│
├── application/
│   ├── service/
│   │   └── TeacherAssignmentService.java
│   ├── dto/
│   │   ├── request/
│   │   │   ├── CreateAssignmentRequest.java
│   │   │   ├── UpdateAssignmentRequest.java
│   │   │   ├── AddClassroomRequest.java
│   │   │   └── AddScheduleRequest.java
│   │   └── response/
│   │       ├── AssignmentResponse.java
│   │       ├── AssignmentDetailResponse.java     ← con classrooms y schedules
│   │       ├── ClassroomAssignmentResponse.java
│   │       └── ScheduleResponse.java
│   └── mapper/
│       └── AssignmentMapper.java
│
├── infrastructure/
│   ├── adapter/
│   │   ├── in/rest/
│   │   │   ├── TeacherAssignmentController.java
│   │   │   │   ├── POST   /api/v1/teacher-assignments
│   │   │   │   ├── GET    /api/v1/teacher-assignments
│   │   │   │   ├── GET    /api/v1/teacher-assignments/{id}
│   │   │   │   ├── GET    /api/v1/teacher-assignments/teacher/{teacherUserId}
│   │   │   │   ├── GET    /api/v1/teacher-assignments/institution/{institutionId}
│   │   │   │   ├── GET    /api/v1/teacher-assignments/status/{status}
│   │   │   │   ├── GET    /api/v1/teacher-assignments/academic-year/{year}
│   │   │   │   ├── PUT    /api/v1/teacher-assignments/{id}
│   │   │   │   ├── PATCH  /api/v1/teacher-assignments/{id}/status
│   │   │   │   ├── DELETE /api/v1/teacher-assignments/{id}
│   │   │   │   └── PATCH  /api/v1/teacher-assignments/{id}/restore
│   │   │   │
│   │   │   ├── AssignmentManagementController.java
│   │   │   │   ├── POST   /api/v1/assignments-management/{id}/classrooms
│   │   │   │   ├── DELETE /api/v1/assignments-management/{id}/classrooms/{classroomId}
│   │   │   │   ├── PATCH  /api/v1/assignments-management/{id}/classrooms/{classroomId}/primary
│   │   │   │   ├── POST   /api/v1/assignments-management/{id}/schedules
│   │   │   │   └── DELETE /api/v1/assignments-management/{id}/schedules/{scheduleId}
│   │   │   │
│   │   │   └── GlobalExceptionHandler.java
│   │   │
│   │   └── out/
│   │       ├── persistence/
│   │       │   ├── entity/
│   │       │   │   ├── TeacherAssignmentEntity.java          ← @Table("teacher_assignments")
│   │       │   │   ├── AssignmentClassroomEntity.java        ← @Table("assignment_classrooms")
│   │       │   │   └── AssignmentScheduleEntity.java         ← @Table("assignment_schedules")
│   │       │   ├── mapper/
│   │       │   │   └── AssignmentPersistenceMapper.java
│   │       │   ├── repository/
│   │       │   │   ├── R2dbcAssignmentRepository.java
│   │       │   │   ├── R2dbcAssignmentClassroomRepository.java
│   │       │   │   └── R2dbcAssignmentScheduleRepository.java
│   │       │   ├── AssignmentPersistenceAdapter.java
│   │       │   ├── ClassroomAssignmentPersistenceAdapter.java
│   │       │   └── SchedulePersistenceAdapter.java
│   │       └── messaging/
│   │           └── RabbitAssignmentEventPublisher.java
│   │
│   ├── client/
│   │   ├── InstitutionClient.java
│   │   ├── UserClient.java
│   │   ├── ClassroomClient.java
│   │   └── CourseClient.java
│   ├── common/
│   │   ├── ApiResponse.java
│   │   └── ErrorResponse.java
│   └── config/
│       ├── R2dbcConfig.java
│       ├── SecurityConfig.java
│       ├── RabbitMQConfig.java
│       └── WebClientConfig.java
│
└── TeacherAssignmentApplication.java

src/main/resources/
├── application.yml
├── application-dev.yml
├── application-vpc.yml
└── db/migration/
    ├── V1__create_teacher_assignments_table.sql
    ├── V2__create_assignment_classrooms_table.sql
    ├── V3__create_assignment_schedules_table.sql
    └── V4__create_assignment_indexes.sql
```

---

### 12. vg-ms-notifications

> **Microservicio de notificaciones vía WhatsApp usando Evolution API.**
> Envía mensajes, archivos, reportes de asistencia, libretas de notas, alertas de incidentes.
> Consume eventos de TODOS los demás MS vía RabbitMQ.
> **Puerto:** 9091 | **BD:** PostgreSQL schema `notifications`

```
src/main/java/pe/edu/vallegrande/sigei/notifications/
│
├── domain/
│   ├── model/
│   │   ├── Notification.java
│   │   │   ├── id: UUID
│   │   │   ├── institutionId: String
│   │   │   ├── recipientId: String              ← userId o guardianId
│   │   │   ├── recipientPhone: String           ← "+51987654321"
│   │   │   ├── recipientName: String
│   │   │   ├── channel: NotificationChannel      ← WHATSAPP (extensible a EMAIL, SMS)
│   │   │   ├── type: NotificationType            ← ATTENDANCE, GRADES, INCIDENT, etc.
│   │   │   ├── templateKey: String               ← "attendance.absent", "grades.report_card"
│   │   │   ├── subject: String
│   │   │   ├── bodyText: String                  ← Texto del mensaje
│   │   │   ├── variables: Map<String, String>    ← Variables para plantilla
│   │   │   ├── attachments: List<Attachment>     ← Archivos adjuntos
│   │   │   ├── status: NotificationStatus        ← PENDING, SENT, DELIVERED, READ, FAILED
│   │   │   ├── retryCount: Integer
│   │   │   ├── maxRetries: Integer               ← default 3
│   │   │   ├── lastError: String
│   │   │   ├── sentAt: LocalDateTime
│   │   │   ├── deliveredAt: LocalDateTime
│   │   │   ├── readAt: LocalDateTime
│   │   │   ├── createdAt: LocalDateTime
│   │   │   └── updatedAt: LocalDateTime
│   │   │
│   │   ├── NotificationTemplate.java
│   │   │   ├── id: UUID
│   │   │   ├── templateKey: String               ← "attendance.absent" (único)
│   │   │   ├── name: String                      ← "Notificación de inasistencia"
│   │   │   ├── type: NotificationType
│   │   │   ├── bodyTemplate: String              ← "Estimado/a {{guardianName}}, le informamos..."
│   │   │   ├── variables: List<String>           ← ["guardianName", "studentName", "date"]
│   │   │   ├── isActive: boolean
│   │   │   ├── createdAt: LocalDateTime
│   │   │   └── updatedAt: LocalDateTime
│   │   │
│   │   ├── Attachment.java                       ← Value Object (record)
│   │   │   ├── fileName: String
│   │   │   ├── mimeType: String                  ← "application/pdf", "image/png"
│   │   │   ├── fileUrl: String                   ← URL del archivo o base64
│   │   │   └── fileSize: Long
│   │   │
│   │   ├── NotificationLog.java
│   │   │   ├── id: UUID
│   │   │   ├── notificationId: UUID
│   │   │   ├── action: String                    ← "SENT", "DELIVERED", "READ", "FAILED", "RETRY"
│   │   │   ├── detail: String
│   │   │   ├── evolutionResponse: String         ← JSON response de Evolution API
│   │   │   └── createdAt: LocalDateTime
│   │   │
│   │   └── enums/
│   │       ├── NotificationChannel.java          ← WHATSAPP, EMAIL, SMS (futuro)
│   │       ├── NotificationType.java             ← ver detalle abajo
│   │       └── NotificationStatus.java           ← PENDING, SENT, DELIVERED, READ, FAILED
│   │
│   ├── port/
│   │   ├── in/
│   │   │   ├── SendNotificationUseCase.java
│   │   │   ├── FindNotificationUseCase.java
│   │   │   ├── RetryNotificationUseCase.java
│   │   │   ├── ManageTemplateUseCase.java
│   │   │   └── ProcessEventUseCase.java          ← Procesa eventos de RabbitMQ
│   │   └── out/
│   │       ├── NotificationRepository.java
│   │       ├── NotificationTemplateRepository.java
│   │       ├── NotificationLogRepository.java
│   │       └── WhatsAppSenderPort.java           ← Interfaz hacia Evolution API
│   │
│   ├── exception/
│   │   ├── NotificationNotFoundException.java
│   │   ├── TemplateNotFoundException.java
│   │   ├── WhatsAppSendException.java
│   │   └── InvalidPhoneNumberException.java
│   │
│   └── event/                        ← Eventos de dominio que CONSUME (desde otros MS vía RabbitMQ)
│       ├── AttendanceAbsentEvent.java         [record] studentId, institutionId, classroomId, date
│       ├── AttendanceLateEvent.java           [record] studentId, institutionId, classroomId, date, arrivalTime
│       ├── ReportCardPublishedEvent.java      [record] reportCardId, studentId, institutionId, academicYear, periodNumber
│       ├── IncidentCreatedEvent.java          [record] incidentId, studentId, institutionId, incidentType, description
│       ├── IncidentResolvedEvent.java         [record] incidentId, studentId, resolution
│       ├── EnrollmentConfirmedEvent.java      [record] enrollmentId, studentId, institutionId, classroomId, academicYear
│       ├── PsychologicalEvaluationCompletedEvent.java [record] evaluationId, studentId, institutionId, evaluationType, requiresFollowUp
│       ├── FollowUpDueEvent.java              [record] evaluationId, studentId, dueDate
│       └── AnnouncementCreatedEvent.java      [record] institutionId, title, message, targetAudience
│
├── application/
│   ├── service/
│   │   ├── NotificationService.java              ← Orquesta envío de notificaciones
│   │   ├── NotificationTemplateService.java      ← CRUD de plantillas
│   │   ├── NotificationRetryService.java         ← Reintentos automáticos
│   │   └── EventProcessorService.java            ← Convierte eventos RabbitMQ → Notification
│   ├── dto/
│   │   ├── request/
│   │   │   ├── SendNotificationRequest.java
│   │   │   ├── SendBulkNotificationRequest.java  ← Envío masivo (ej: todas las faltas del día)
│   │   │   ├── CreateTemplateRequest.java
│   │   │   └── UpdateTemplateRequest.java
│   │   └── response/
│   │       ├── NotificationResponse.java
│   │       ├── NotificationDetailResponse.java   ← con logs
│   │       ├── NotificationStatsResponse.java    ← estadísticas
│   │       └── TemplateResponse.java
│   ├── mapper/
│   │   ├── NotificationMapper.java
│   │   └── TemplateMapper.java
│   └── event/                                    ← DTOs de eventos que recibe de otros MS
│       ├── AttendanceEvent.java                  ← Evento de asistencia (de MS Assistance)
│       ├── GradePublishedEvent.java              ← Libreta publicada (de MS Notes)
│       ├── IncidentCreatedEvent.java             ← Incidente (de MS Disciplinary)
│       ├── EnrollmentConfirmedEvent.java         ← Matrícula confirmada (de MS Enrollments)
│       ├── EvaluationCompletedEvent.java         ← Eval. psicológica (de MS Psychology)
│       └── InstitutionAnnouncementEvent.java     ← Comunicado general (de MS Institution)
│
├── infrastructure/
│   ├── adapter/
│   │   ├── in/
│   │   │   ├── rest/
│   │   │   │   ├── NotificationController.java
│   │   │   │   │   ├── POST   /api/v1/notifications/send
│   │   │   │   │   ├── POST   /api/v1/notifications/send-bulk
│   │   │   │   │   ├── GET    /api/v1/notifications
│   │   │   │   │   ├── GET    /api/v1/notifications/{id}
│   │   │   │   │   ├── GET    /api/v1/notifications/{id}/logs
│   │   │   │   │   ├── GET    /api/v1/notifications/recipient/{recipientId}
│   │   │   │   │   ├── GET    /api/v1/notifications/institution/{institutionId}
│   │   │   │   │   ├── GET    /api/v1/notifications/status/{status}
│   │   │   │   │   ├── GET    /api/v1/notifications/stats/{institutionId}
│   │   │   │   │   └── POST   /api/v1/notifications/{id}/retry
│   │   │   │   │
│   │   │   │   ├── TemplateController.java
│   │   │   │   │   ├── GET    /api/v1/templates
│   │   │   │   │   ├── GET    /api/v1/templates/{id}
│   │   │   │   │   ├── GET    /api/v1/templates/key/{templateKey}
│   │   │   │   │   ├── POST   /api/v1/templates
│   │   │   │   │   ├── PUT    /api/v1/templates/{id}
│   │   │   │   │   └── DELETE /api/v1/templates/{id}
│   │   │   │   │
│   │   │   │   ├── WebhookController.java        ← Recibe webhooks de Evolution API
│   │   │   │   │   └── POST   /api/v1/webhooks/evolution
│   │   │   │   │
│   │   │   │   └── GlobalExceptionHandler.java
│   │   │   │
│   │   │   └── messaging/                        ← Adaptadores de ENTRADA (RabbitMQ consumers)
│   │   │       ├── AttendanceEventListener.java
│   │   │       │   └── @RabbitListener("notification.attendance")
│   │   │       ├── GradeEventListener.java
│   │   │       │   └── @RabbitListener("notification.grades")
│   │   │       ├── IncidentEventListener.java
│   │   │       │   └── @RabbitListener("notification.incidents")
│   │   │       ├── EnrollmentEventListener.java
│   │   │       │   └── @RabbitListener("notification.enrollments")
│   │   │       ├── PsychologyEventListener.java
│   │   │       │   └── @RabbitListener("notification.psychology")
│   │   │       └── AnnouncementEventListener.java
│   │   │           └── @RabbitListener("notification.announcements")
│   │   │
│   │   └── out/
│   │       ├── persistence/
│   │       │   ├── entity/
│   │       │   │   ├── NotificationEntity.java       ← @Table("notifications")
│   │       │   │   ├── NotificationTemplateEntity.java ← @Table("notification_templates")
│   │       │   │   └── NotificationLogEntity.java    ← @Table("notification_logs")
│   │       │   ├── mapper/
│   │       │   │   ├── NotificationPersistenceMapper.java
│   │       │   │   ├── TemplatePersistenceMapper.java
│   │       │   │   └── LogPersistenceMapper.java
│   │       │   ├── repository/
│   │       │   │   ├── R2dbcNotificationRepository.java
│   │       │   │   ├── R2dbcTemplateRepository.java
│   │       │   │   └── R2dbcLogRepository.java
│   │       │   ├── NotificationPersistenceAdapter.java
│   │       │   ├── TemplatePersistenceAdapter.java
│   │       │   └── LogPersistenceAdapter.java
│   │       │
│   │       └── whatsapp/                         ← Adaptador hacia EVOLUTION API
│   │           ├── EvolutionApiClient.java        ← WebClient → Evolution API
│   │           ├── EvolutionWhatsAppAdapter.java  ← implements WhatsAppSenderPort
│   │           ├── dto/
│   │           │   ├── EvolutionSendTextRequest.java
│   │           │   ├── EvolutionSendMediaRequest.java
│   │           │   ├── EvolutionSendDocumentRequest.java
│   │           │   ├── EvolutionWebhookPayload.java
│   │           │   └── EvolutionResponse.java
│   │           └── mapper/
│   │               └── EvolutionMapper.java       ← Domain → Evolution API format
│   │
│   ├── client/
│   │   ├── StudentClient.java                    ← Obtener datos del estudiante y guardián
│   │   ├── InstitutionClient.java                ← Datos de la institución
│   │   └── UserClient.java                       ← Datos del usuario para el teléfono
│   ├── common/
│   │   ├── ApiResponse.java
│   │   └── ErrorResponse.java
│   └── config/
│       ├── R2dbcConfig.java
│       ├── SecurityConfig.java
│       ├── RabbitMQConfig.java
│       ├── EvolutionApiConfig.java               ← URL, API Key, Instance Name
│       ├── WebClientConfig.java
│       └── SchedulerConfig.java                  ← Para reintentos programados
│
└── NotificationsApplication.java

src/main/resources/
├── application.yml
├── application-dev.yml
├── application-vpc.yml
└── db/migration/
    ├── V1__create_notifications_table.sql
    ├── V2__create_notification_templates_table.sql
    ├── V3__create_notification_logs_table.sql
    ├── V4__insert_default_templates.sql
    └── V5__create_notification_indexes.sql
```

#### Tipos de Notificación (NotificationType)

```java
public enum NotificationType {
    // ── Asistencia ──
    ATTENDANCE_ABSENT,          // "Su hijo/a no asistió hoy"
    ATTENDANCE_LATE,            // "Su hijo/a llegó tarde"
    ATTENDANCE_DAILY_SUMMARY,   // Resumen diario de asistencia del aula

    // ── Notas / Evaluaciones ──
    GRADES_REPORT_CARD,         // Libreta de notas publicada (enviar PDF)
    GRADES_EVALUATION,          // Nueva evaluación registrada

    // ── Disciplina / Incidentes ──
    INCIDENT_CREATED,           // Nuevo incidente reportado
    INCIDENT_RESOLVED,          // Incidente resuelto
    BEHAVIOR_ALERT,             // Alerta de comportamiento

    // ── Psicología ──
    PSYCHOLOGY_EVALUATION,      // Evaluación psicológica completada
    PSYCHOLOGY_FOLLOW_UP,       // Recordatorio de seguimiento

    // ── Matrículas ──
    ENROLLMENT_CONFIRMED,       // Matrícula confirmada
    ENROLLMENT_PERIOD_OPEN,     // Período de matrícula abierto

    // ── Institucional ──
    ANNOUNCEMENT,               // Comunicado general de la institución
    EVENT_REMINDER,             // Recordatorio de evento cívico/escolar

    // ── Sistema ──
    CUSTOM                      // Mensaje personalizado
}
```

#### Plantillas de Mensaje (ejemplos)

```
─────────────────────────────────────────────────────────
Template Key: attendance.absent
─────────────────────────────────────────────────────────
🏫 *{{institutionName}}*

Estimado/a *{{guardianName}}*,

Le informamos que su hijo/a *{{studentName}}* del aula
*{{classroomName}}* no asistió a clases el día *{{date}}*.

Si tiene alguna justificación, acérquese a la institución
o comuníquese con la docente.

_Mensaje automático — SIGEI_

─────────────────────────────────────────────────────────
Template Key: grades.report_card
─────────────────────────────────────────────────────────
🏫 *{{institutionName}}*

Estimado/a *{{guardianName}}*,

La libreta de notas de *{{studentName}}* correspondiente
al *{{periodName}}* del año *{{academicYear}}* ha sido
publicada.

📄 Se adjunta el documento en PDF.

Si tiene consultas, comuníquese con la docente del aula
*{{classroomName}}*.

_Mensaje automático — SIGEI_

─────────────────────────────────────────────────────────
Template Key: incident.created
─────────────────────────────────────────────────────────
🏫 *{{institutionName}}*

⚠️ Estimado/a *{{guardianName}}*,

Le informamos que se ha registrado un incidente relacionado
con su hijo/a *{{studentName}}* el día *{{date}}*.

*Tipo:* {{incidentType}}
*Descripción:* {{description}}
*Acción tomada:* {{actionTaken}}

Por favor, acérquese a la institución para coordinar
el seguimiento.

_Mensaje automático — SIGEI_
```

#### Evolution API — Configuración

```yaml
# application.yml — sección de Evolution API
evolution:
  api:
    base-url: ${EVOLUTION_API_URL:http://localhost:8085}
    api-key: ${EVOLUTION_API_KEY}
    instance-name: ${EVOLUTION_INSTANCE:sigei-whatsapp}

  webhook:
    url: ${EVOLUTION_WEBHOOK_URL:http://ms-notifications:9091/api/v1/webhooks/evolution}
    events:
      - MESSAGES_UPSERT           # Mensaje enviado/recibido
      - MESSAGES_UPDATE           # Estado actualizado (delivered, read)
      - CONNECTION_UPDATE         # Estado de conexión WhatsApp
```

#### Evolution API — Endpoints que consume el MS

```
Evolution API Base: http://evolution-api:8085

POST /message/sendText/{instance}         ← Enviar mensaje de texto
POST /message/sendMedia/{instance}        ← Enviar imagen/video/audio
POST /message/sendWhatsAppAudio/{instance}
POST /message/sendDocument/{instance}     ← Enviar PDF (libretas, reportes)
POST /message/sendSticker/{instance}
GET  /instance/connectionState/{instance} ← Verificar si WhatsApp está conectado
POST /instance/create                     ← Crear instancia de WhatsApp
GET  /instance/fetchInstances             ← Listar instancias
```

---

### 13. vg-ms-gateway

> API Gateway — Punto de entrada único. Enruta requests, valida JWT de Keycloak, aplica CORS.
> **Puerto:** 8080

```
src/main/java/pe/edu/vallegrande/sigei/gateway/
│
├── config/
│   ├── CorsConfig.java                          ← CORS centralizado (ÚNICO lugar)
│   ├── SecurityConfig.java                       ← OAuth2 Resource Server + Keycloak JWT
│   ├── RouteConfig.java                          ← Rutas a todos los MS
│   ├── RateLimitConfig.java                      ← Rate limiting por IP/token
│   └── CircuitBreakerConfig.java                 ← Resilience4j fallbacks
│
├── filter/
│   ├── AuthenticationFilter.java                 ← Valida JWT en cada request
│   ├── LoggingFilter.java                        ← Log de requests entrantes
│   └── RateLimitFilter.java
│
└── GatewayApplication.java

src/main/resources/
├── application.yml
│   spring:
│     cloud:
│       gateway:
│         routes:
│           - id: ms-institution
│             uri: lb://MS-INSTITUTION
│             predicates:
│               - Path=/api/v1/institutions/**, /api/v1/classrooms/**
│           - id: ms-students
│             uri: lb://MS-STUDENTS
│             predicates:
│               - Path=/api/v1/students/**, /api/v1/guardians/**
│           - id: ms-enrollments
│             uri: lb://MS-ENROLLMENTS
│             predicates:
│               - Path=/api/v1/enrollments/**, /api/v1/academic-periods/**
│           - id: ms-users
│             uri: lb://MS-USERS
│             predicates:
│               - Path=/api/v1/users/**
│           - id: ms-academic
│             uri: lb://MS-ACADEMIC
│             predicates:
│               - Path=/api/v1/courses/**, /api/v1/competencies/**,
│                       /api/v1/capacities/**, /api/v1/performances/**,
│                       /api/v1/catalog/**
│           - id: ms-civic-dates
│             uri: lb://MS-CIVIC-DATES
│             predicates:
│               - Path=/api/v1/events/**, /api/v1/calendars/**
│           - id: ms-notes
│             uri: lb://MS-NOTES
│             predicates:
│               - Path=/api/v1/evaluations/**, /api/v1/report-cards/**
│           - id: ms-assistance
│             uri: lb://MS-ASSISTANCE
│             predicates:
│               - Path=/api/v1/attendance/**, /api/v1/attendance-summary/**
│           - id: ms-disciplinary
│             uri: lb://MS-DISCIPLINARY
│             predicates:
│               - Path=/api/v1/behavior-records/**, /api/v1/incidents/**
│           - id: ms-psychology
│             uri: lb://MS-PSYCHOLOGY
│             predicates:
│               - Path=/api/v1/psychological-evaluations/**,
│                       /api/v1/special-needs-support/**
│           - id: ms-teacher-assignment
│             uri: lb://MS-TEACHER-ASSIGNMENT
│             predicates:
│               - Path=/api/v1/teacher-assignments/**,
│                       /api/v1/assignments-management/**
│           - id: ms-notifications
│             uri: lb://MS-NOTIFICATIONS
│             predicates:
│               - Path=/api/v1/notifications/**, /api/v1/templates/**
├── application-dev.yml
└── application-vpc.yml
```

---

### 14. vg-ms-eureka-server

> Service Discovery — Registro de todos los microservicios.
> **Puerto:** 8761

```
src/main/java/pe/edu/vallegrande/sigei/eureka/
│
└── EurekaServerApplication.java                  ← @EnableEurekaServer

src/main/resources/
├── application.yml
│   server:
│     port: 8761
│   eureka:
│     client:
│       register-with-eureka: false
│       fetch-registry: false
│     server:
│       enable-self-preservation: false
└── application-vpc.yml
```

---

## 📊 MAPA DE COMUNICACIÓN ENTRE MICROSERVICIOS

```
                                    ┌──────────────┐
                          ┌────────►│ INSTITUTION  │◄─────────────────────────┐
                          │         │    :9080     │                          │
                          │         └──────────────┘                          │
                          │              ▲  ▲  ▲                              │
              ┌───────────┤              │  │  │                              │
              │           │    ┌─────────┘  │  └──────────┐                  │
              │           │    │            │              │                  │
         ┌────┴─────┐ ┌──┴────┴──┐  ┌──────┴───┐  ┌──────┴──────┐  ┌──────┴──────┐
         │ STUDENTS │ │ENROLLMENTS│  │  USERS   │  │  ACADEMIC   │  │ CIVIC DATES │
         │  :9081   │ │  :9082   │  │  :9083   │  │   :9084     │  │   :9085     │
         └────┬─────┘ └──────────┘  └──────┬───┘  └─────────────┘  └─────────────┘
              │                             │
    ┌─────────┼─────────┬─────────┬────────┤
    │         │         │         │        │
┌───┴────┐ ┌─┴──────┐ ┌┴────────┐│ ┌──────┴──────┐
│ NOTES  │ │ASSIST. │ │DISCIPL. ││ │  TEACHER    │
│ :9086  │ │ :9087  │ │ :9088   ││ │  ASSIGN.    │
└────────┘ └────────┘ └─────────┘│ │   :9099     │
                                  │ └─────────────┘
                          ┌───────┴───────┐
                          │  PSYCHOLOGY   │
                          │    :9090      │
                          └───────────────┘

                    ─── RabbitMQ (eventos) ───▼

                          ┌───────────────┐
                          │ NOTIFICATIONS │ ◄── Consume eventos de TODOS
                          │    :9091      │
                          │  (Evolution)  │ ──► WhatsApp API
                          └───────────────┘
```

### Comunicación síncrona (WebClient) — Quién llama a quién

| MS que llama | MS que consulta |
|---|---|
| Students | Institution, Users |
| Enrollments | Institution, Students |
| Users | Institution |
| Academic | Institution |
| Civic Dates | Institution |
| Notes | Institution, Students, Academic |
| Assistance | Institution, Students |
| Disciplinary | Institution, Students, Users |
| Psychology | Institution, Students, Users |
| Teacher Assignment | Institution, Users |

### Comunicación asíncrona (RabbitMQ) — Eventos hacia Notifications

| MS que publica | Evento | Cola destino |
|---|---|---|
| Assistance | `attendance.absent`, `attendance.late` | `notification.attendance` |
| Notes | `grades.report_published` | `notification.grades` |
| Disciplinary | `incident.created`, `incident.resolved` | `notification.incidents` |
| Enrollments | `enrollment.confirmed` | `notification.enrollments` |
| Psychology | `evaluation.completed`, `follow_up.due` | `notification.psychology` |
| Institution | `announcement.created` | `notification.announcements` |

---

## 📏 REGLAS DE NOMENCLATURA

| Elemento | Convención | Ejemplo |
|----------|-----------|---------|
| Paquete base | `pe.edu.vallegrande.sigei.<modulo>` | `pe.edu.vallegrande.sigei.institution` |
| Entidad de dominio | PascalCase, sin sufijos | `Institution`, `Student` |
| Entidad de persistencia | PascalCase + `Entity` | `InstitutionEntity` |
| Repository (dominio) | `<Nombre>Repository` | `InstitutionRepository` |
| Repository (R2DBC) | `R2dbc<Nombre>Repository` | `R2dbcInstitutionRepository` |
| Adapter de persistencia | `<Nombre>PersistenceAdapter` | `InstitutionPersistenceAdapter` |
| Use Case (puerto in) | `<Verbo><Nombre>UseCase` | `CreateInstitutionUseCase` |
| Service (implementación) | `<Nombre>Service` | `InstitutionService` |
| Controller | `<Nombre>Controller` | `InstitutionController` |
| Mapper (aplicación) | `<Nombre>Mapper` | `InstitutionMapper` |
| Mapper (persistencia) | `<Nombre>PersistenceMapper` | `InstitutionPersistenceMapper` |
| DTO request | `<Verbo><Nombre>Request` | `CreateInstitutionRequest` |
| DTO response | `<Nombre>Response` | `InstitutionResponse` |
| Excepción not found | `<Nombre>NotFoundException` | `InstitutionNotFoundException` |
| Tabla BD | snake_case, plural | `institutions`, `attendance_records` |
| Migración Flyway | `V<N>__<descripcion>.sql` | `V1__create_institutions_table.sql` |
| Endpoint base | `/api/v1/<recurso>` | `/api/v1/institutions` |

---

## 🔗 RELACIÓN CON OTROS DOCUMENTOS

| Documento | Relación |
|-----------|----------|
| [01_ARQUITECTURA_HEXAGONAL](01_ARQUITECTURA_HEXAGONAL_CORRECTA.md) | Define los principios que esta estructura implementa |
| [02_COMUNICACION](02_COMUNICACION_SINCRONA_ASINCRONA.md) | Detalle de WebClient (sync) y RabbitMQ (async) |
| [03_BASE_DE_DATOS](03_BASE_DE_DATOS_RECOMENDACION.md) | PostgreSQL + schema-per-service + Flyway |
| [04_API_GATEWAY](04_API_GATEWAY_Y_SERVICE_DISCOVERY.md) | Config detallada del Gateway + Eureka |
| [05_ARQUITECTURA_BACKEND](05_ARQUITECTURA_BACKEND_COMPLETA.md) | Código de cada capa (dominio, application, infra) |
| [08_SEGURIDAD_KEYCLOAK](08_SEGURIDAD_KEYCLOAK.md) | SecurityConfig.java en cada MS |
| [09_API_RESPONSE](09_API_RESPONSE_Y_ERROR_RESPONSE.md) | ApiResponse + ErrorResponse en `infrastructure/common/` |
| [10_DESPLIEGUE_VPC](10_DESPLIEGUE_VPC.md) | Docker Compose y deploy de todos estos MS |
| [11_COMUNICACION_CAPAS](11_COMUNICACION_ENTRE_CAPAS.md) | Cómo fluye una request entre las carpetas |
