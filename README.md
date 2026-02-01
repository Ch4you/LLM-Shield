# LLM-Shield

```mermaid
graph TD
    %% --- Global Styles ---
    classDef default font-family:Arial,fontSize:13px;
    classDef goService fill:#00ADD8,stroke:#005c73,stroke-width:2px,color:white,rx:5,ry:5;
    classDef db fill:#A41E11,stroke:#5c0e07,stroke-width:2px,color:white,rx:5,ry:5;
    classDef queue fill:#231F20,stroke:#5c5c5c,stroke-width:2px,color:white,rx:5,ry:5;
    classDef ai fill:#FF9900,stroke:#b36b00,stroke-width:2px,color:white,rx:5,ry:5;
    classDef client fill:#2E7D32,stroke:#1b5e20,stroke-width:2px,color:white,rx:5,ry:5;
    classDef cluster fill:#f5f7fa,stroke:#cfd8dc,stroke-width:2px,stroke-dasharray: 5 5;

    %% --- Actors ---
    User((Client / User)):::client

    %% --- System Boundary ---
    subgraph "LLM-Shield System (Private Network)"
        direction TB

        %% Gateway Layer
        subgraph "Gateway Layer (Go/Gin)"
            direction TB
            API_GW[API Gateway<br/>(Validation / Auth)]:::goService
            RateLimiter[Rate Limiter]:::goService
            
            ProxyLogic[Proxy Controller<br/>(Business Logic)]:::goService
        end

        %% Data Processing Layer
        subgraph "Vector Engine"
            direction TB
            VectorAdapter[Vector Adapter]:::goService
            GC_Worker[LRU Garbage Collector]:::goService
        end
        
        %% Async Layer
        subgraph "Audit Service"
            direction TB
            KafkaProducer[Log Producer]:::goService
            KafkaConsumer[Audit Consumer]:::goService
        end
    end

    %% --- Infrastructure Layer ---
    subgraph "Data & Messaging Infrastructure"
        Redis[("Redis<br/>(Hot Cache / Rate Limit / Auth)")]:::db
        Qdrant[("Qdrant<br/>(Vector Storage)")]:::db
        Kafka[("Apache Kafka<br/>(Event Bus)")]:::queue
        DynamoDB[("AWS DynamoDB<br/>(Audit Logs)")]:::db
    end

    %% --- External AI Services ---
    subgraph "Model Inference Layer"
        EmbModel["Nomic-Embed-Text<br/>(Embedding Model)"]:::ai
        ChatModel["Llama 3.2<br/>(Inference Engine)"]:::ai
    end

    %% --- Relationships ---

    %% 1. Ingress
    User ==>|HTTP/JSON| API_GW

    %% 2. Gateway Processing
    API_GW -->|1. Check Token/Limit| Redis
    API_GW --> RateLimiter
    RateLimiter --> ProxyLogic

    %% 3. Proxy Logic Flow
    ProxyLogic -->|2. Semantic Search| VectorAdapter
    VectorAdapter -->|RPC| EmbModel
    VectorAdapter -->|Query (Cosine)| Qdrant
    VectorAdapter -.->|Hit: Update Score| Redis

    %% 4. Miss / Inference Flow
    VectorAdapter -- "Cache Miss" --> ProxyLogic
    ProxyLogic -->|3. Forward Request| ChatModel
    ProxyLogic -.->|Async Save| VectorAdapter
    VectorAdapter -.->|Write Vector| Qdrant
    VectorAdapter -.->|Write ZSET| Redis

    %% 5. Audit Flow (Decoupled)
    ProxyLogic -.->|4. Push Event| KafkaProducer
    KafkaProducer ==>|Topic: logs| Kafka
    Kafka ==>|Subscribe| KafkaConsumer
    KafkaConsumer -->|Persist| DynamoDB

    %% 6. Maintenance
    GC_Worker -.->|Monitor & Prune| Redis
    GC_Worker -.->|Delete Vectors| Qdrant

    %% --- Links Styling ---
    linkStyle default stroke-width:2px,fill:none,stroke:#546e7a;

```