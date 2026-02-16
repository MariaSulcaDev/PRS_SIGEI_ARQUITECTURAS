# 11 — COMUNICACIÓN ENTRE CAPAS (Arquitectura Hexagonal)

> **Objetivo:** Explicar cómo fluye una petición a través de las capas de la arquitectura hexagonal, cómo se comunican entre sí, y por qué las dependencias van siempre hacia adentro.
> **Prerrequisito:** Haber leído [01_ARQUITECTURA_HEXAGONAL_CORRECTA.md](01_ARQUITECTURA_HEXAGONAL_CORRECTA.md)

---

## 🎯 LAS 3 CAPAS Y SU PROPÓSITO

```
┌─────────────────────────────────────────────────────────────┐
│                    INFRASTRUCTURE                           │
│                (Adaptadores — el "cómo")                    │
│                                                             │
│   Adaptadores de ENTRADA         Adaptadores de SALIDA     │
│   ┌───────────────────┐         ┌───────────────────────┐  │
│   │ REST Controller   │         │ Persistence Adapter   │  │
│   │ (recibe HTTP)     │         │ (guarda en BD)        │  │
│   └────────┬──────────┘         └───────────▲───────────┘  │
│            │                                │               │
│ ═══════════╪════════════════════════════════╪═══════════════│
│            │         APPLICATION            │               │
│            │      (Orquestación)            │               │
│            ▼                                │               │
│   ┌───────────────────┐                    │               │
│   │ Service           │                    │               │
│   │ (orquesta el      │────────────────────┘               │
│   │  caso de uso)     │                                    │
│   └────────┬──────────┘                                    │
│            │                                               │
│ ═══════════╪═══════════════════════════════════════════════ │
│            │          DOMAIN                               │
│            │     (Reglas de negocio)                        │
│            ▼                                               │
│   ┌───────────────────┐    ┌───────────────────┐          │
│   │ Model/Entity      │    │ Port (Interface)  │          │
│   │ (Institution,     │    │ (contratos)       │          │
│   │  Student, etc.)   │    │                   │          │
│   └───────────────────┘    └───────────────────┘          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 🔑 REGLA DE ORO: Las dependencias van hacia ADENTRO

```
Infrastructure → depende de → Application → depende de → Domain

Domain NO depende de NADIE (es el núcleo puro)
```

Esto significa:

- **Domain** no importa nada de Spring, ni de MongoDB, ni de PostgreSQL.
- **Application** solo conoce interfaces (ports) del dominio.
- **Infrastructure** implementa todo lo concreto (BD, HTTP, mensajería).

---

## 📡 FLUJO COMPLETO DE UNA PETICIÓN

### Ejemplo: `POST /api/institutions` — Crear una institución

```
PASO 1                    PASO 2                    PASO 3
────────────────         ────────────────          ────────────────
INFRASTRUCTURE           APPLICATION               DOMAIN
(Adaptador IN)           (Service)                 (Modelo)

HTTP Request  ───►  InstitutionController
                    │
                    │ Convierte DTO → Domain
                    │ (usando Mapper)
                    ▼
              CreateInstitutionUseCase  ◄── Puerto de ENTRADA (interface)
                    │
                    │ InstitutionService
                    │ (implementa el caso de uso)
                    │
                    ├─► Institution.create(...)  ◄── Lógica de dominio
                    │   (valida código modular,     (reglas de negocio)
                    │    nombre, etc.)
                    │
                    ├─► InstitutionRepository   ◄── Puerto de SALIDA (interface)
                    │   .save(institution)          (definido en dominio)
                    │
              ──────┼──────────────────────────────
                    │
PASO 4              ▼
────────────────
INFRASTRUCTURE
(Adaptador OUT)

InstitutionPersistenceAdapter
    │ implementa InstitutionRepository
    │
    │ Convierte Domain → Document
    │ (usando PersistenceMapper)
    ▼
MongoInstitutionRepository.save(document)
    │
    ▼
 MongoDB / PostgreSQL
```

---

## 🧩 CÓDIGO PASO A PASO — Cómo se comunica cada capa

### PASO 1: Controller (Infrastructure → Application)

```java
// CAPA: infrastructure/adapter/in/rest/
// FUNCIÓN: Recibir HTTP, convertir DTO, delegar al servicio

@RestController
@RequestMapping("/api/institutions")
public class InstitutionController {

    // ⚡ El controller NO conoce InstitutionService directamente.
    //    Conoce la INTERFAZ (port) del dominio.
    private final CreateInstitutionUseCase createUseCase;
    private final InstitutionMapper mapper;

    public InstitutionController(CreateInstitutionUseCase createUseCase,
                                  InstitutionMapper mapper) {
        this.createUseCase = createUseCase;
        this.mapper = mapper;
    }

    @PostMapping
    public Mono<ResponseEntity<ApiResponse<InstitutionResponse>>> create(
            @Valid @RequestBody CreateInstitutionRequest request) {

        // 1️⃣ Recibir el DTO del request HTTP
        // 2️⃣ Convertir DTO → Objeto de dominio
        Institution institution = mapper.toDomain(request);

        // 3️⃣ Delegar al CASO DE USO (puerto de entrada)
        return createUseCase.execute(institution)
            .map(mapper::toResponse)  // 6️⃣ Convertir Domain → Response DTO
            .map(resp -> ResponseEntity
                .status(HttpStatus.CREATED)
                .body(ApiResponse.created(resp, "Institución creada")));
    }
}
```

**¿Qué conoce el Controller?**

- ✅ `CreateInstitutionUseCase` (interfaz del dominio)
- ✅ `CreateInstitutionRequest` / `InstitutionResponse` (DTOs de aplicación)
- ✅ `ApiResponse` (wrapper de infraestructura)
- ❌ NO conoce `InstitutionService` (la implementación concreta)
- ❌ NO conoce `MongoInstitutionRepository` ni `InstitutionDocument`

---

### PASO 2: Puerto de Entrada — Use Case (Dominio define el contrato)

```java
// CAPA: domain/port/in/
// FUNCIÓN: Definir QUÉ se puede hacer (no CÓMO)

public interface CreateInstitutionUseCase {

    /**
     * Crea una nueva institución educativa.
     * @param institution la entidad de dominio ya construida
     * @return la institución creada con ID asignado
     */
    Mono<Institution> execute(Institution institution);
}
```

**¿Por qué es una interfaz?**

- El dominio dice "necesito poder crear instituciones" (QUÉ)
- La capa de aplicación decide CÓMO implementarlo
- El controller solo conoce esta interfaz, no la clase concreta

---

### PASO 3: Service — Implementa el Use Case (Application)

```java
// CAPA: application/service/
// FUNCIÓN: Orquestar la lógica, coordinar dominio + puertos de salida

@Service
public class InstitutionService implements CreateInstitutionUseCase {

    // ⚡ El service conoce los PUERTOS DE SALIDA (interfaces),
    //    no las implementaciones concretas.
    private final InstitutionRepository repository;   // ← Puerto de salida (interfaz)
    private final EventPublisher eventPublisher;       // ← Puerto de salida (interfaz)

    public InstitutionService(InstitutionRepository repository,
                               EventPublisher eventPublisher) {
        this.repository = repository;
        this.eventPublisher = eventPublisher;
    }

    @Override
    public Mono<Institution> execute(Institution institution) {

        // 4️⃣ Verificar regla de negocio: no duplicar código modular
        return repository.findByModularCode(institution.getModularCode())
            .flatMap(existing -> Mono.<Institution>error(
                new DuplicateModularCodeException(institution.getModularCode())))

            // 5️⃣ Guardar usando el puerto de salida
            .switchIfEmpty(repository.save(institution))

            // 5b. Publicar evento de dominio (asíncrono)
            .doOnSuccess(saved ->
                eventPublisher.publish(new InstitutionCreatedEvent(
                    saved.getId(), saved.getName())));
    }
}
```

**¿Qué conoce el Service?**

- ✅ `Institution` (modelo de dominio)
- ✅ `InstitutionRepository` (interfaz del dominio — puerto de salida)
- ✅ `EventPublisher` (interfaz del dominio — puerto de salida)
- ✅ Excepciones de dominio (`DuplicateModularCodeException`)
- ❌ NO conoce `MongoInstitutionRepository`, `InstitutionDocument`, ni `@Document`
- ❌ NO conoce `InstitutionController`, `ApiResponse`, ni HTTP

---

### PASO 4: Puerto de Salida — Repository (Dominio define el contrato)

```java
// CAPA: domain/port/out/
// FUNCIÓN: Definir QUÉ necesita el dominio de persistencia (no CÓMO)

public interface InstitutionRepository {

    Mono<Institution> save(Institution institution);
    Mono<Institution> findById(String id);
    Mono<Institution> findByModularCode(String modularCode);
    Flux<Institution> findAll();
    Mono<Void> deleteById(String id);
}
```

**¿Por qué está en el dominio?**

- El dominio dice "necesito guardar y buscar instituciones" (QUÉ)
- La infraestructura decide si usa MongoDB, PostgreSQL, API externa, etc. (CÓMO)
- Si mañana cambias de MongoDB a PostgreSQL, el dominio NO se modifica

---

### PASO 5: Adaptador de Persistencia (Infrastructure implementa el puerto)

```java
// CAPA: infrastructure/adapter/out/persistence/
// FUNCIÓN: Implementar el puerto de salida usando tecnología concreta

@Component
public class InstitutionPersistenceAdapter implements InstitutionRepository {

    // ⚡ Aquí SÍ hay dependencias de tecnología (Spring Data, MongoDB/R2DBC)
    private final MongoInstitutionRepository mongoRepository;
    private final InstitutionPersistenceMapper mapper;

    public InstitutionPersistenceAdapter(
            MongoInstitutionRepository mongoRepository,
            InstitutionPersistenceMapper mapper) {
        this.mongoRepository = mongoRepository;
        this.mapper = mapper;
    }

    @Override
    public Mono<Institution> save(Institution institution) {
        // Convertir Domain → Document (entidad de persistencia)
        InstitutionDocument document = mapper.toDocument(institution);

        // Guardar usando Spring Data
        return mongoRepository.save(document)
            // Convertir Document → Domain
            .map(mapper::toDomain);
    }

    @Override
    public Mono<Institution> findById(String id) {
        return mongoRepository.findById(id)
            .map(mapper::toDomain);
    }

    @Override
    public Mono<Institution> findByModularCode(String modularCode) {
        return mongoRepository.findByModularCode(modularCode)
            .map(mapper::toDomain);
    }

    @Override
    public Flux<Institution> findAll() {
        return mongoRepository.findAll()
            .map(mapper::toDomain);
    }

    @Override
    public Mono<Void> deleteById(String id) {
        return mongoRepository.deleteById(id);
    }
}
```

**¿Qué conoce el Adapter?**

- ✅ `InstitutionRepository` (interfaz del dominio que implementa)
- ✅ `Institution` (modelo del dominio)
- ✅ `MongoInstitutionRepository` (Spring Data — tecnología)
- ✅ `InstitutionDocument` (entidad con `@Document` — tecnología)
- ✅ `InstitutionPersistenceMapper` (convierte Domain ↔ Document)
- ❌ NO conoce al Service ni al Controller

---

## 🔄 DIAGRAMA DE DEPENDENCIAS (Imports)

```
┌──────────────────────────────────────────────────────────────────┐
│                                                                  │
│  InstitutionController                                          │
│  ├── import CreateInstitutionUseCase  ←── (domain/port/in)      │
│  ├── import InstitutionMapper         ←── (application/mapper)   │
│  ├── import CreateInstitutionRequest  ←── (application/dto)      │
│  ├── import InstitutionResponse       ←── (application/dto)      │
│  └── import ApiResponse               ←── (infrastructure)       │
│                                                                  │
│  InstitutionService                                              │
│  ├── import CreateInstitutionUseCase  ←── (domain/port/in)      │
│  ├── import InstitutionRepository     ←── (domain/port/out)     │
│  ├── import Institution               ←── (domain/model)         │
│  └── import DuplicateModularCodeEx.   ←── (domain/exception)    │
│                                                                  │
│  InstitutionPersistenceAdapter                                   │
│  ├── import InstitutionRepository     ←── (domain/port/out)     │
│  ├── import Institution               ←── (domain/model)         │
│  ├── import InstitutionDocument       ←── (infrastructure)       │
│  └── import MongoInstitutionRepo      ←── (infrastructure)       │
│                                                                  │
│  Institution (DOMINIO)                                           │
│  └── import NADA externo              ←── (0 dependencias)      │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

**Observa:**

- `Institution.java` NO importa nada de Spring, MongoDB, ni R2DBC.
- `InstitutionService` solo importa interfaces (ports) y modelos del dominio.
- Solo `InstitutionPersistenceAdapter` importa cosas de MongoDB/Spring Data.

---

## 🔌 ¿CÓMO SE CONECTAN? — Inyección de Dependencias (Spring)

Spring Boot conecta todo automáticamente gracias a `@Component`, `@Service`, etc.:

```
Spring IoC Container:
│
├── Busca: ¿Quién implementa CreateInstitutionUseCase?
│   └── Encuentra: InstitutionService (@Service)
│       └── Inyecta en InstitutionController
│
├── Busca: ¿Quién implementa InstitutionRepository?
│   └── Encuentra: InstitutionPersistenceAdapter (@Component)
│       └── Inyecta en InstitutionService
│
└── Resultado:
    Controller → Service → PersistenceAdapter
    (pero cada uno SOLO conoce la INTERFAZ del anterior)
```

```java
// Spring ve esto en tiempo de ejecución:

// 1. Controller pide CreateInstitutionUseCase
//    → Spring le da InstitutionService

// 2. InstitutionService pide InstitutionRepository
//    → Spring le da InstitutionPersistenceAdapter

// 3. InstitutionPersistenceAdapter pide MongoInstitutionRepository
//    → Spring Data crea una implementación automática
```

---

## 🗺️ MAPPERS — Los traductores entre capas

Cada capa tiene su propia representación de los datos. Los Mappers traducen entre ellos:

```
HTTP Request                Domain Model              Persistence Document
(DTO de entrada)           (Objeto puro)              (Entidad de BD)

CreateInstitutionRequest  →  Institution             →  InstitutionDocument
{                            {                           {
  "name": "IE ...",           id: null,                   _id: ObjectId(...),
  "modularCode": "123",      modularCode: "1234567",     modularCode: "1234567",
  "address": {...}            name: "IE ...",             name: "IE ...",
}                             status: ACTIVE,             status: "ACTIVE",
                              createdAt: now()            createdAt: ISODate(...)
                            }                           }

         ▲                        ▲                         ▲
         │                        │                         │
   InstitutionMapper       (objeto en memoria)     PersistenceMapper
   (application/mapper)                            (infrastructure/mapper)
         │                        │                         │
         ▼                        ▼                         ▼

InstitutionResponse     ←  Institution             ←  InstitutionDocument
{                            (retorno)                  (lectura de BD)
  "id": "abc-123",
  "name": "IE ...",
  "status": "ACTIVE"
}
```

### Código de los Mappers

```java
// ─── Mapper de Aplicación (DTO ↔ Domain) ───
// CAPA: application/mapper/

@Component
public class InstitutionMapper {

    /** DTO Request → Domain Model */
    public Institution toDomain(CreateInstitutionRequest request) {
        return Institution.create(
            request.modularCode(),
            request.name(),
            Address.of(request.department(), request.province(), request.district())
        );
    }

    /** Domain Model → DTO Response */
    public InstitutionResponse toResponse(Institution institution) {
        return new InstitutionResponse(
            institution.getId(),
            institution.getModularCode(),
            institution.getName(),
            institution.getStatus().name(),
            institution.getCreatedAt()
        );
    }
}

// ─── Mapper de Persistencia (Domain ↔ Document) ───
// CAPA: infrastructure/adapter/out/persistence/mapper/

@Component
public class InstitutionPersistenceMapper {

    /** Domain Model → Persistence Document */
    public InstitutionDocument toDocument(Institution institution) {
        InstitutionDocument doc = new InstitutionDocument();
        doc.setId(institution.getId());
        doc.setModularCode(institution.getModularCode());
        doc.setName(institution.getName());
        doc.setStatus(institution.getStatus().name());
        doc.setCreatedAt(institution.getCreatedAt());
        doc.setUpdatedAt(institution.getUpdatedAt());
        return doc;
    }

    /** Persistence Document → Domain Model */
    public Institution toDomain(InstitutionDocument doc) {
        return Institution.reconstitute(    // ← factory diferente, sin validaciones
            doc.getId(),
            doc.getModularCode(),
            doc.getName(),
            InstitutionStatus.valueOf(doc.getStatus()),
            doc.getCreatedAt(),
            doc.getUpdatedAt()
        );
    }
}
```

---

## ❓ PREGUNTAS FRECUENTES

### ¿Por qué no pongo @Document directo en Institution.java?

```
❌ INCORRECTO (lo que tienen hoy):
@Document(collection = "institutions")
public class Institution {
    @Id
    private String id;
    ...
}

Problema: El dominio "sabe" que usa MongoDB.
Si cambias a PostgreSQL, tienes que modificar tu modelo de negocio.
```

```
✅ CORRECTO (hexagonal):
// domain/model/Institution.java — CERO anotaciones de BD
public class Institution {
    private String id;
    ...
}

// infrastructure/persistence/InstitutionDocument.java — aquí van las anotaciones
@Document(collection = "institutions")
public class InstitutionDocument {
    @Id
    private String id;
    ...
}
```

### ¿Controller puede llamar directo al Repository?

```
❌ NUNCA:
Controller → Repository     (salta la lógica de negocio)

✅ SIEMPRE:
Controller → UseCase → Service → Repository
```

Si el controller llama directo al repository, estás haciendo una API CRUD sin reglas de negocio. Cualquier validación se pierde.

### ¿El Service puede retornar un DTO?

```
❌ INCORRECTO:
public Mono<InstitutionResponse> execute(...) {
    // El service conoce DTOs de HTTP → acoplamiento
}

✅ CORRECTO:
public Mono<Institution> execute(...) {
    // El service retorna objetos de DOMINIO
    // El controller/mapper convierte a DTO
}
```

### ¿Dónde pongo las validaciones?

```
┌────────────────────────────────────────────────────────────┐
│ TIPO DE VALIDACIÓN           │ DÓNDE VA                   │
├──────────────────────────────┼────────────────────────────│
│ Formato (email, longitud)    │ DTO Request (@Valid)        │
│ Regla de negocio simple      │ Domain Model (constructor)  │
│ Regla que necesita BD        │ Application Service         │
│ (ej: "código no duplicado")  │ (usa Repository para        │
│                              │  verificar)                 │
└────────────────────────────────────────────────────────────┘
```

---

## 📊 RESUMEN VISUAL — Quién conoce a quién

```
                    CONOCE                  NO CONOCE
                    ──────                  ─────────
Controller      →   UseCase (interfaz)      Service (clase concreta)
                    DTO Request/Response    Document, @Document
                    ApiResponse             MongoDB, PostgreSQL
                    Mapper de aplicación

Service         →   Domain Model            Controller
                    Port In (interfaz)      DTO Request/Response
                    Port Out (interfaz)     ApiResponse
                    Excepciones dominio     @Document, @Table

PersistenceAdap →   Port Out (interfaz)     Controller
                    Domain Model            Service
                    Document/Entity         DTO Request/Response
                    Spring Data Repository  ApiResponse

Domain Model    →   NADA externo            Spring, MongoDB, R2DBC
                    Solo Java puro          HTTP, JSON, REST
                    Value Objects propios   Annotations de frameworks
```

---

## 🔗 RELACIÓN CON OTROS DOCUMENTOS

| Documento | Relación |
|-----------|----------|
| [01_ARQUITECTURA_HEXAGONAL](01_ARQUITECTURA_HEXAGONAL_CORRECTA.md) | Define la estructura completa de la hexagonal |
| [05_ARQUITECTURA_BACKEND](05_ARQUITECTURA_BACKEND_COMPLETA.md) | Estructura de carpetas que refleja estas capas |
| [07_PATRONES_DISEÑO](07_PATRONES_DISENO_RECOMENDADOS.md) | Patrones que se aplican dentro de cada capa |
| [09_API_RESPONSE](09_API_RESPONSE_Y_ERROR_RESPONSE.md) | ApiResponse vive SOLO en infrastructure, el dominio no lo conoce |
