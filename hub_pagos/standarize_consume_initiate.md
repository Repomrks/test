# Standarize Consumer Initiate - Diagramas de Arquitectura

## 1. Flow Diagram

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
