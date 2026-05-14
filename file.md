# Standarize Consumer Initiate - Diagramas de Arquitectura

## Indice y Resumen

Este documento contiene 13 diagramas que describen la arquitectura completa del microservicio **Standarize Consumer Initiate**, un procesador de instrucciones de pago desplegado como AWS Lambda y activado por eventos Kafka.

| # | Diagrama | Que refleja |
|---|---|---|
| 1 | **Flow Diagram** | Vision general del flujo de datos desde la recepcion del evento Kafka hasta la respuesta final, mostrando las dos ramas de procesamiento (archivo y manual) y el manejo de errores con notificacion. |
| 2 | **Component Diagram** | Estructura de componentes organizada por capas (entry point, application, domain, ports, adapters, servicios externos), mostrando las dependencias entre ellos y como se conectan a la infraestructura. |
| 3 | **Sequence Diagram** | Interaccion temporal paso a paso entre todos los participantes del sistema para ambos flujos: procesamiento de archivo (3.1) con descarga S3, validacion y batch insert; y procesamiento manual (3.2) con construccion de pago desde DTO. |
| 4 | **Infrastructure Diagram** | Topologia de infraestructura AWS en produccion: Lambda dentro de VPC con subnets privadas, PostgreSQL, S3, Kafka self-managed, Parameter Store, y el pipeline CI/CD con GitLab. Incluye el esquema de base de datos y sus relaciones. |
| 5 | **Error Handling Flow** | Secuencia detallada de como se manejan los distintos tipos de error (storage, validacion, base de datos, publicacion Kafka), mostrando que siempre se publica un evento de error antes de re-lanzar la excepcion. |
| 6 | **Batch Processing Detail** | Mecanismo interno del repositorio para insertar grandes volumenes: verificacion de idempotencia, resolucion de referencias (producto, canal, convenio con cache), mapeo a entidades ORM, e insercion en batches de 500 con 4 operaciones concurrentes. |
| 7 | **Validation Rules Detail** | Arbol de decision completo del servicio de validacion de dominio: validacion estructural (array, longitud), validacion por registro (campos requeridos, monto numerico positivo, formato de fecha, unicidad de payment_id), y acumulacion de errores. |
| 8 | **NestJS Module Dependency Graph** | Grafo de dependencias entre todos los modulos NestJS del proyecto: HandlerModule como raiz, ConfigifyModule para secrets, y el feature module con sus sub-modulos (Database, Kafka, S3) y sus providers internos. |
| 9 | **Database Entity Relationship Diagram** | Modelo entidad-relacion completo de PostgreSQL con las 5 tablas (product, channel, agreement, payment_instruction, historical_payment), sus columnas, tipos de datos, claves primarias, foraneas y relaciones de cardinalidad. |
| 10 | **Cold Start vs Warm Start** | Comparacion secuencial entre la primera invocacion (cold start: creacion de contexto NestJS, conexion a DB, conexion a Kafka, cache de app) y las invocaciones subsiguientes (warm start: retorno inmediato de la instancia cacheada). |
| 11 | **Event Message Format** | Transformacion del mensaje a traves del sistema: evento Kafka de entrada (base64 encoded), payload decodificado con sus campos, y los dos posibles eventos de salida (exito con status OK, error con status ERROR y mensaje). |
| 12 | **Idempotency Strategy** | Estrategia de idempotencia en dos niveles: nivel archivo (lookup del primer payment_id para evitar re-procesamiento completo) y nivel registro (UNIQUE INDEX en payment_reference con orIgnore para manejar race conditions sin fallar). |
| 13 | **Docker Compose Local Environment** | Topologia del entorno de desarrollo local con Docker Compose: contenedores (Lambda, LocalStack, Kafka, Zookeeper, Kafka UI, PostgreSQL), scripts de inicializacion, health checks, y orden de arranque con dependencias. |

---

## 1. Flow Diagram
```mermaid

```

```mermaid
flowchart TD
    A[Kafka Event - SelfManagedKafkaEvent] --> B[Lambda Handler - main.ts]
    B --> C[Extract and Decode - Base64 to JSON]
    C --> D[Parser Service - Validate DTO]
    D --> E{eventType?}

    E -->|file| F[StandarizeConsumerFileUseCase]
    E -->|manual| G[StandarizeConsumerManualUseCase]

    F --> F1[Download file from S3]
    F1 --> F2[Validate JSON structure]
    F2 --> F3[Validate payments - required fields, amounts, dates]
    F3 --> F4[Idempotency check - payment_reference exists?]
    F4 -->|duplicate| F5[Return: Duplicated file]
    F4 -->|new| F6[Batch insert to PostgreSQL - 500 records/batch, 4 concurrent]
    F6 --> F7[Publish success event to Kafka]
    F7 --> F8[Return: N payments processed]

    G --> G1[Build PaymentInstruction from header]
    G1 --> G2[Validate single payment]
    G2 --> G3[Save to PostgreSQL]
    G3 --> G4[Publish success event to Kafka]
    G4 --> G5[Return: Manual payment processed]

    F1 -.->|error| ERR[Publish error event to Kafka]
    F2 -.->|error| ERR
    F3 -.->|error| ERR
    F6 -.->|error| ERR
    G2 -.->|error| ERR
    G3 -.->|error| ERR
    ERR --> THROW[Re-throw exception]
```

## 2. Component Diagram

```mermaid
graph TB
    subgraph Entry_Point[Entry Point]
        LAMBDA[Lambda Handler - main.ts]
        HANDLER[HandlerController]
    end

    subgraph Application_Layer[Application Layer]
        FILE_UC[StandarizeConsumerFileUseCase]
        MANUAL_UC[StandarizeConsumerManualUseCase]
        EVENT_SVC[EventNotificationService]
    end

    subgraph Domain_Layer[Domain Layer]
        VALIDATOR[PaymentValidationService]
    end

    subgraph Ports[Ports - Interfaces]
        STORAGE_PORT[StoragePort]
        REPO_PORT[PaymentInstructionRepositoryPort]
        EVENT_PORT[EventProducerPort]
    end

    subgraph Infrastructure_Adapters[Infrastructure Adapters]
        S3_ADAPTER[StorageAdapter - AWS S3]
        DB_ADAPTER[PaymentInstructionRepositoryAdapter - TypeORM]
        KAFKA_ADAPTER[EventProducerAdapter - Kafka Producer]
    end

    subgraph External_Services[External Services]
        S3[(AWS S3 - payment-bucket)]
        PG[(PostgreSQL - payment_instruction)]
        KAFKA[[Apache Kafka - payment-topic]]
    end

    LAMBDA --> HANDLER
    HANDLER --> FILE_UC
    HANDLER --> MANUAL_UC

    FILE_UC --> STORAGE_PORT
    FILE_UC --> REPO_PORT
    FILE_UC --> EVENT_SVC
    FILE_UC --> VALIDATOR

    MANUAL_UC --> REPO_PORT
    MANUAL_UC --> EVENT_SVC
    MANUAL_UC --> VALIDATOR

    EVENT_SVC --> EVENT_PORT

    STORAGE_PORT -.-> S3_ADAPTER
    REPO_PORT -.-> DB_ADAPTER
    EVENT_PORT -.-> KAFKA_ADAPTER

    S3_ADAPTER --> S3
    DB_ADAPTER --> PG
    KAFKA_ADAPTER --> KAFKA
```

## 3. Sequence Diagram

### 3.1 File Flow

```mermaid
sequenceDiagram
    participant K as Kafka
    participant L as Lambda Handler
    participant P as Parser Service
    participant H as HandlerController
    participant UC as FileUseCase
    participant S3 as StorageAdapter - S3
    participant V as PaymentValidationService
    participant DB as PaymentInstructionRepository
    participant EN as EventNotificationService
    participant KP as Kafka Producer

    K->>L: SelfManagedKafkaEvent
    L->>L: extractKafkaRecordValue base64 decode
    L->>P: parse rawValue DTO
    P-->>L: StandarizeConsumerInitiateRequestDto
    L->>H: handlerStadarizeConsumer body
    H->>UC: execute FileProcessCommand

    UC->>S3: download key fileName
    S3-->>UC: JSON content

    UC->>UC: Validate JSON structure
    UC->>V: validate payments array
    V->>V: validateStructure array and length max 100k
    V->>V: validateRecords fields amounts dates duplicates
    V-->>UC: valid

    UC->>DB: saveAll payments fileName productCode channelCode
    DB->>DB: Idempotency check first payment_reference
    DB->>DB: Lookup product by code
    DB->>DB: Lookup channel by code
    DB->>DB: Lookup agreements by code cached
    DB->>DB: Batch insert 500 per batch 4 concurrent
    DB-->>UC: number array saved IDs

    UC->>EN: notifySuccess transactionId channelCode
    EN->>KP: send topic EventMessage status OK
    KP-->>EN: ack

    UC-->>H: message transactionId totalPayments
    H-->>L: result
    L-->>K: Lambda response
```

### 3.2 Manual Flow

```mermaid
sequenceDiagram
    participant K as Kafka
    participant L as Lambda Handler
    participant P as Parser Service
    participant H as HandlerController
    participant UC as ManualUseCase
    participant V as PaymentValidationService
    participant DB as PaymentInstructionRepository
    participant EN as EventNotificationService
    participant KP as Kafka Producer

    K->>L: SelfManagedKafkaEvent
    L->>L: extractKafkaRecordValue base64 decode
    L->>P: parse rawValue DTO
    P-->>L: StandarizeConsumerInitiateRequestDto
    L->>H: handlerStadarizeConsumer body
    H->>UC: execute ManualProcessCommand

    UC->>UC: buildPayment transactionId header
    UC->>V: validate single payment
    V-->>UC: valid

    UC->>DB: saveAll payment manual productCode channelCode
    DB-->>UC: number array saved IDs

    UC->>EN: notifySuccess transactionId channelCode
    EN->>KP: send topic EventMessage status OK
    KP-->>EN: ack

    UC-->>H: message transactionId
    H-->>L: result
    L-->>K: Lambda response
```

## 4. Infrastructure Diagram

```mermaid
graph TB
    subgraph AWS_Cloud[AWS Cloud]
        subgraph VPC[VPC]
            subgraph Private_Subnets[Private Subnets]
                LAMBDA[AWS Lambda<br/>lambda-dev-standarize-initiate<br/>Node.js 22.x - 128MB - 3s timeout]
            end

            subgraph Data_Layer[Data Layer]
                RDS[(PostgreSQL<br/>payment_instruction<br/>product channel<br/>agreement historical_payment)]
            end
        end

        S3[(S3 Bucket<br/>payment-bucket<br/>Payment JSON files)]

        subgraph Event_Streaming[Event Streaming]
            MSK[Apache Kafka<br/>Topic: payment-topic<br/>Group: payment-group]
        end

        SSM[AWS Systems Manager<br/>Parameter Store and Secrets Manager]
    end

    MSK -->|Trigger SelfManagedKafka| LAMBDA
    LAMBDA -->|Download files| S3
    LAMBDA -->|Read and Write| RDS
    LAMBDA -->|Produce events| MSK
    LAMBDA -->|Fetch config| SSM

    subgraph CICD[CI/CD]
        GITLAB[GitLab CI/CD]
        ZIP[build.zip artifact]
    end

    GITLAB --> ZIP --> LAMBDA

    subgraph Database_Schema[Database Schema]
        direction LR
        T1[product<br/>id name code enabled]
        T2[channel<br/>id name code service_description]
        T3[agreement<br/>id code name product_id client_rut]
        T4[payment_instruction<br/>id payer_rut amount currency<br/>beneficiary status agreement_id<br/>product_id channel_id]
        T5[historical_payment<br/>id payment_id status_code]
    end

    T1 --- T3
    T1 --- T4
    T2 --- T4
    T3 --- T4
    T4 --- T5
```

---

### Resumen de Tecnologias

| Componente | Tecnologia |
|---|---|
| Runtime | Node.js 22.x (AWS Lambda) |
| Framework | NestJS (Application Context) |
| Base de datos | PostgreSQL (TypeORM) |
| Storage | AWS S3 |
| Mensajeria | Apache Kafka (Self-Managed) |
| Configuracion | AWS Parameter Store / Secrets Manager |
| CI/CD | GitLab CI/CD |
| Contenedor build | Docker (Alpine + zip) |

---

## 5. Error Handling Flow

```mermaid
sequenceDiagram
    participant K as Kafka
    participant L as Lambda Handler
    participant UC as FileUseCase
    participant S3 as StorageAdapter
    participant V as PaymentValidationService
    participant DB as PaymentInstructionRepository
    participant EN as EventNotificationService
    participant KP as Kafka Producer

    K->>L: SelfManagedKafkaEvent
    L->>UC: execute FileProcessCommand

    alt Storage Error
        UC->>S3: download key fileName
        S3--xUC: StorageException - File not found
        UC->>EN: notifyError transactionId channelCode error message
        EN->>KP: send topic EventMessage status ERROR
        KP-->>EN: ack
        UC--xL: throw StorageException
    else Validation Error
        UC->>S3: download key fileName
        S3-->>UC: JSON content
        UC->>V: validate payments array
        V--xUC: ValidationException - missing fields or invalid data
        UC->>EN: notifyError transactionId channelCode error message
        EN->>KP: send topic EventMessage status ERROR
        KP-->>EN: ack
        UC--xL: throw ValidationException
    else Database Error
        UC->>S3: download key fileName
        S3-->>UC: JSON content
        UC->>V: validate payments array
        V-->>UC: valid
        UC->>DB: saveAll payments
        DB--xUC: DatabaseException - Product or Channel not found
        UC->>EN: notifyError transactionId channelCode error message
        EN->>KP: send topic EventMessage status ERROR
        KP-->>EN: ack
        UC--xL: throw DatabaseException
    else Kafka Publish Error
        UC->>EN: notifySuccess transactionId channelCode
        EN->>KP: send topic EventMessage
        KP--xEN: EventPublishException - Broker unavailable
        EN--xUC: throw EventPublishException
        UC--xL: throw EventPublishException
    end

    L-->>K: Error response
```

## 6. Batch Processing Detail

```mermaid
sequenceDiagram
    participant UC as FileUseCase
    participant DB as PaymentInstructionRepository
    participant PG as PostgreSQL

    UC->>DB: saveAll 10000 payments fileName productCode channelCode

    Note over DB: Step 1 - Idempotency Check
    DB->>PG: SELECT id FROM payment_instruction WHERE payment_reference = first_id
    PG-->>DB: null - not found proceed

    Note over DB: Step 2 - Resolve References
    DB->>PG: SELECT id FROM product WHERE code = productCode
    PG-->>DB: product.id
    DB->>PG: SELECT id FROM channel WHERE code = channelCode
    PG-->>DB: channel.id

    Note over DB: Step 3 - Resolve Agreements with Cache
    loop For each unique agreement_code
        DB->>PG: SELECT id FROM agreement WHERE code = agreementCode
        PG-->>DB: agreement.id
        DB->>DB: Cache agreement_code to id
    end

    Note over DB: Step 4 - Map to ORM Entities
    DB->>DB: Build 10000 OrmPaymentInstruction entities

    Note over DB: Step 5 - Batch Insert with Concurrency
    rect rgb(240, 248, 255)
        Note over DB,PG: Batch 1-4 concurrent - 500 records each
        par Batch 1
            DB->>PG: INSERT 500 records orIgnore returning id
        and Batch 2
            DB->>PG: INSERT 500 records orIgnore returning id
        and Batch 3
            DB->>PG: INSERT 500 records orIgnore returning id
        and Batch 4
            DB->>PG: INSERT 500 records orIgnore returning id
        end
        PG-->>DB: generated IDs batch 1-4
    end

    rect rgb(240, 248, 255)
        Note over DB,PG: Batch 5-8 concurrent - 500 records each
        par Batch 5
            DB->>PG: INSERT 500 records orIgnore returning id
        and Batch 6
            DB->>PG: INSERT 500 records orIgnore returning id
        and Batch 7
            DB->>PG: INSERT 500 records orIgnore returning id
        and Batch 8
            DB->>PG: INSERT 500 records orIgnore returning id
        end
        PG-->>DB: generated IDs batch 5-8
    end

    Note over DB: ... continues until all 20 batches processed

    DB-->>UC: number array all saved IDs
```

## 7. Validation Rules Detail

```mermaid
flowchart TD
    START[payments field from JSON] --> A{Is Array?}
    A -->|No| ERR1[ValidationException: payments must be an array]
    A -->|Yes| B{Length > 0?}
    B -->|No| ERR2[ValidationException: File contains no payments]
    B -->|Yes| C{Length <= 100000?}
    C -->|No| ERR3[ValidationException: Exceeds maximum 100k payments]
    C -->|Yes| D[For each payment record]

    D --> E{Is valid object?}
    E -->|No| ERR4[Payment N: not a valid object]
    E -->|Yes| F[Check required fields]

    F --> G{All required present?}
    G -->|No| ERR5[Payment N: required fields missing: field1, field2]
    G -->|Yes| H{payment_amount numeric?}

    H -->|No| ERR6[Payment N: amount must be numeric]
    H -->|Yes| I{amount finite?}
    I -->|No| ERR7[Payment N: amount is not a valid number]
    I -->|Yes| J{amount > 0?}
    J -->|No| ERR8[Payment N: amount must be greater than 0]
    J -->|Yes| K{execution_date YYYY-MM-DD?}

    K -->|No| ERR9[Payment N: execution_date must be YYYY-MM-DD]
    K -->|Yes| L{payment_id unique in file?}
    L -->|No| ERR10[Payment N: payment_id duplicated in file]
    L -->|Yes| VALID[Record valid - continue next]

    ERR1 --> THROW[Throw ValidationException]
    ERR2 --> THROW
    ERR3 --> THROW
    ERR4 --> COLLECT[Collect all errors]
    ERR5 --> COLLECT
    ERR6 --> COLLECT
    ERR7 --> COLLECT
    ERR8 --> COLLECT
    ERR9 --> COLLECT
    ERR10 --> COLLECT
    COLLECT --> THROW

    subgraph Required_Fields[Required Fields]
        direction LR
        RF1[payment_id]
        RF2[payer_rut]
        RF3[payer_name]
        RF4[payer_account_number]
        RF5[payment_amount]
        RF6[payment_currency]
        RF7[execution_date]
        RF8[beneficiary_name]
        RF9[beneficiary_account_number]
        RF10[beneficiary_rut]
    end
```

## 8. NestJS Module Dependency Graph

```mermaid
graph TD
    subgraph Root[HandlerModule]
        HC[HandlerController]
    end

    subgraph Config[ConfigifyModule]
        direction TB
        CF1[SecretsDBConfiguration]
        CF2[SecretsKafkaConfiguration]
        CF3[SecretsS3Configuration]
    end

    subgraph Parser[ParserModule]
        PS[ValidationPipeParserService]
    end

    subgraph Feature[StandarizeConsumerInitiateModule]
        FUC[StandarizeConsumerFileUseCase]
        MUC[StandarizeConsumerManualUseCase]
        ENS[EventNotificationService]
        PVS[PaymentValidationService]
    end

    subgraph DB[DatabaseModule]
        direction TB
        TORM[TypeOrmModule.forRoot - PostgreSQL]
        REPO[PaymentInstructionRepositoryAdapter]
        ENT1[OrmPaymentInstruction]
        ENT2[OrmProduct]
        ENT3[OrmChannel]
        ENT4[OrmAgreement]
    end

    subgraph Kafka[KafkaModule]
        direction TB
        CM[ClientsModule.registerAsync - Kafka Transport]
        EPA[EventProducerAdapter]
    end

    subgraph Storage[S3Module]
        direction TB
        S3C[S3Client]
        SA[StorageAdapter]
    end

    Root --> Config
    Root --> Parser
    Root --> Feature

    Feature --> DB
    Feature --> Kafka
    Feature --> Storage

    HC --> FUC
    HC --> MUC

    FUC --> SA
    FUC --> REPO
    FUC --> ENS
    FUC --> PVS

    MUC --> REPO
    MUC --> ENS
    MUC --> PVS

    ENS --> EPA

    DB --> CF1
    Kafka --> CF2
    Storage --> CF3

    TORM --> ENT1
    TORM --> ENT2
    TORM --> ENT3
    TORM --> ENT4
```

## 9. Database Entity Relationship Diagram

```mermaid
erDiagram
    PRODUCT {
        int id PK
        varchar name
        boolean enabled
        varchar code UK
        timestamptz created_at
        timestamptz updated_at
    }

    CHANNEL {
        int id PK
        varchar name
        varchar service_description
        varchar code UK
        timestamptz created_at
        timestamptz updated_at
    }

    AGREEMENT {
        int id PK
        varchar code UK
        varchar name
        int product_id FK
        varchar client_rut
        timestamptz date_agreement
        boolean enabled
        timestamptz created_at
        timestamptz updated_at
    }

    PAYMENT_INSTRUCTION {
        int id PK
        varchar payer_rut
        varchar payer_name
        varchar payer_account_number
        numeric payment_amount
        char payment_currency
        varchar payment_purpose
        varchar service_description
        date execution_date
        varchar payment_reference UK
        varchar beneficiary_name
        varchar beneficiary_account_number
        varchar beneficiary_rut
        varchar destination_financial_entity
        varchar payment_reason
        varchar payment_status
        varchar file_name
        varchar status_code
        varchar reason_code
        int agreement_id FK
        int product_id FK
        int channel_id FK
        varchar iban
        varchar bic
        varchar address_line1
        varchar address_line2
        char country_code
        timestamptz created_at
        timestamptz updated_at
    }

    HISTORICAL_PAYMENT {
        int id PK
        int payment_id FK
        varchar payer_rut
        varchar payer_name
        varchar payer_account_number
        varchar status_code
        varchar reason_code
        timestamptz created_at
        timestamptz updated_at
    }

    PRODUCT ||--o{ AGREEMENT : has
    PRODUCT ||--o{ PAYMENT_INSTRUCTION : categorizes
    CHANNEL ||--o{ PAYMENT_INSTRUCTION : routes
    AGREEMENT ||--o{ PAYMENT_INSTRUCTION : governs
    PAYMENT_INSTRUCTION ||--o{ HISTORICAL_PAYMENT : tracks
```

## 10. Cold Start vs Warm Start

```mermaid
sequenceDiagram
    participant AWS as AWS Lambda Runtime
    participant BS as bootstrap()
    participant NEST as NestJS Context
    participant CFG as ConfigifyModule
    participant DB as TypeORM Connection
    participant KFK as Kafka Client

    Note over AWS,KFK: COLD START - First Invocation

    AWS->>BS: bootstrap()
    BS->>NEST: NestFactory.createApplicationContext(HandlerModule)
    NEST->>CFG: Initialize ConfigifyModule
    CFG->>CFG: Resolve secrets from Parameter Store
    CFG->>CFG: Validate configuration classes
    NEST->>DB: TypeOrmModule.forRoot - Create connection pool
    DB->>DB: Connect to PostgreSQL
    NEST->>KFK: ClientsModule.registerAsync - Create Kafka client
    KFK->>KFK: Connect to Kafka brokers
    NEST-->>BS: App context ready
    BS->>BS: Cache app instance in module scope
    BS-->>AWS: Return app

    Note over AWS,KFK: WARM START - Subsequent Invocations

    AWS->>BS: bootstrap()
    BS->>BS: Check if app exists - YES
    BS-->>AWS: Return cached app immediately

    Note over AWS: No re-initialization needed
    Note over AWS: DB connections reused
    Note over AWS: Kafka client reused
    Note over AWS: Config already resolved
```

## 11. Event Message Format

```mermaid
flowchart LR
    subgraph Input_Event[Input - SelfManagedKafkaEvent]
        direction TB
        IE1[eventSource: SelfManagedKafka]
        IE2[bootstrapServers: broker:29092]
        IE3[records: topic-partition array]
        IE4[record.value: base64 encoded JSON]
    end

    subgraph Decoded_Payload[Decoded Payload]
        direction TB
        DP1[timestamp: ISO 8601]
        DP2[transactionId: UUID]
        DP3[eventType: file or manual]
        DP4[product_code: TEF PAP NOM TIN PAY]
        DP5[channel_code: WEB H2H API]
        DP6[payload: FilePayload or ManualPayload]
    end

    subgraph Output_Success[Output Event - Success]
        direction TB
        OS1[timestamp: ISO 8601]
        OS2[hashId: transactionId]
        OS3[channelId: channelCode]
        OS4[eventType: standarize-consume]
        OS5["payload: { status: OK }"]
    end

    subgraph Output_Error[Output Event - Error]
        direction TB
        OE1[timestamp: ISO 8601]
        OE2[hashId: transactionId]
        OE3[channelId: channelCode]
        OE4[eventType: standarize-consume]
        OE5["payload: { status: ERROR, error: message }"]
    end

    Input_Event -->|base64 decode + JSON parse| Decoded_Payload
    Decoded_Payload -->|processing success| Output_Success
    Decoded_Payload -->|processing failure| Output_Error
```

## 12. Idempotency Strategy

```mermaid
flowchart TD
    START[saveAll called with payments array] --> A[Extract first payment_id as reference]
    A --> B{Query: EXISTS payment_reference?}

    B -->|Found in DB| C[Return empty array]
    C --> D[Use Case detects empty result]
    D --> E[Return: Duplicated file - 0 payments processed]
    E --> F[No error event published]

    B -->|Not found| G[Proceed with insert]
    G --> H[Build ORM entities]
    H --> I[Batch INSERT with orIgnore]

    I --> J{Individual record duplicate?}
    J -->|Duplicate payment_reference| K[orIgnore skips silently]
    J -->|New record| L[Insert succeeds]

    K --> M[Continue with next records]
    L --> M
    M --> N[Return array of generated IDs]

    subgraph Strategy[Two-Level Idempotency]
        direction TB
        S1[Level 1: File-level check<br/>First payment_id lookup<br/>Prevents full re-processing]
        S2[Level 2: Record-level check<br/>UNIQUE INDEX on payment_reference<br/>orIgnore handles race conditions]
    end
```

## 13. Docker Compose Local Environment

```mermaid
graph TB
    subgraph Docker_Network[Docker Compose Network]
        subgraph App[Lambda Container - port 9000]
            LAMBDA[NestJS Lambda<br/>Runtime Interface Emulator]
        end

        subgraph Storage_Layer[Storage]
            LS[LocalStack - port 4566<br/>S3 Service]
        end

        subgraph Messaging[Messaging]
            ZK[Zookeeper - port 2181]
            BROKER[Kafka Broker - port 9092]
            KUI[Kafka UI - port 8080]
        end

        subgraph Database[Database]
            PG[PostgreSQL 16 - port 5432]
        end
    end

    LAMBDA -->|GetObject| LS
    LAMBDA -->|Produce messages| BROKER
    LAMBDA -->|TypeORM queries| PG
    BROKER --> ZK
    KUI --> BROKER

    subgraph Init_Scripts[Initialization on Startup]
        direction TB
        I1[s3-init.sh<br/>Create bucket + upload test files]
        I2[kafka-init.sh<br/>Create payment-topic]
        I3[init.sql<br/>Create tables and indexes]
        I4[seed.sql<br/>Insert products channels agreements]
    end

    I1 -.-> LS
    I2 -.-> BROKER
    I3 -.-> PG
    I4 -.-> PG

    subgraph Health_Checks[Health Checks]
        HC1[LocalStack: curl localhost:4566/_localstack/health]
        HC2[PostgreSQL: pg_isready -U postgres]
    end

    HC1 -.-> LS
    HC2 -.-> PG

    subgraph Depends_On[Startup Order]
        direction LR
        DO1[ZK starts first] --> DO2[Broker starts]
        DO3[PG healthy] --> DO4[Lambda starts]
        DO5[LS healthy] --> DO4
        DO2 --> DO4
    end
```
