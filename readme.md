# SpurrinAI Backend - Architecture Documentation

## Table of Contents
1. [System Overview](#system-overview)
2. [High-Level Architecture](#high-level-architecture)
3. [Component Architecture](#component-architecture)
4. [Data Flow Architecture](#data-flow-architecture)
5. [Technology Stack](#technology-stack)
6. [Database Architecture](#database-architecture)
7. [Security Architecture](#security-architecture)
8. [Deployment Architecture](#deployment-architecture)

---

## System Overview

SpurrinAI is an intelligent document processing and question-answering system that leverages RAG (Retrieval Augmented Generation) architecture. The system processes medical documents, creates vector embeddings, and provides real-time AI-powered responses through a hybrid microservices architecture.

### Key Capabilities
- **Document Processing**: PDF ingestion and intelligent chunking
- **Vector Search**: ChromaDB-based semantic search with Redis caching
- **Real-time Chat**: WebSocket-based communication with AI responses
- **Multi-tenancy**: Hospital-based data isolation
- **AI Integration**: Natural language processing with external AI services

---

## High-Level Architecture

```mermaid
graph TB
    subgraph "Client Layer"
        WEB[Web Application]
        MOBILE[Mobile App]
    end
    
    subgraph "API Gateway Layer"
        LB[Load Balancer]
        NGINX[Nginx Reverse Proxy]
    end
    
    subgraph "Application Layer"
        API[REST API Service]
        AI[AI Processing Service]
        WS[WebSocket Server]
    end
    
    subgraph "Data Layer"
        MYSQL[(MySQL Database)]
        REDIS[(Redis Cache)]
        CHROMA[(ChromaDB Vector Store)]
        FILES[File Storage]
    end
    
    subgraph "External Services"
        OPENAI[OpenAI API]
        EMAIL[Email Service]
    end
    
    WEB --> LB
    MOBILE --> LB
    LB --> NGINX
    NGINX --> API
    NGINX --> WS
    API --> AI
    WS --> AI
    API --> MYSQL
    API --> REDIS
    AI --> MYSQL
    AI --> REDIS
    AI --> CHROMA
    AI --> OPENAI
    API --> EMAIL
    API --> FILES
```

---

## Component Architecture

### 1. Primary API Service

```mermaid
graph TB
    subgraph "API Application"
        APP[Main Application Entry Point]
        
        subgraph "Middleware Layer"
            SEC[Security Middleware]
            AUTH[Authentication]
            RATE[Rate Limiting]
            CORS[CORS Handler]
            LOG[Logging]
        end
        
        subgraph "Route Layer"
            ROUTES[Route Handlers]
            AUTH_R[Auth Routes]
            HOSP_R[Hospital Routes]
            USER_R[User Routes]
            DOC_R[Document Routes]
            ADMIN_R[Admin Routes]
        end
        
        subgraph "Controller Layer"
            CTRL[Controllers]
            AUTH_C[Auth Controller]
            HOSP_C[Hospital Controller]
            USER_C[User Controller]
            DOC_C[Document Controller]
        end
        
        subgraph "Service Layer"
            SERV[Services]
            TOKEN_S[Token Service]
            HOSP_S[Hospital Service]
            USER_S[User Service]
            CRON_S[Cron Jobs]
        end
        
        subgraph "Data Layer"
            DB[Database Layer]
            CONFIG[Configuration]
        end
    end
    
    APP --> SEC
    SEC --> AUTH
    AUTH --> RATE
    RATE --> CORS
    CORS --> LOG
    LOG --> ROUTES
    ROUTES --> CTRL
    CTRL --> SERV
    SERV --> DB
```

### 2. AI Processing Service (NLP/RAG Engine)

```mermaid
graph TB
    subgraph "AI Processing Application"
        AI_APP[AI Service Entry Point]
        
        subgraph "NLP Processing"
            NLP[Natural Language Processing]
            TEXT[Text Processing]
            EMBED[Embedding Generation]
        end
        
        subgraph "Vector Operations"
            CHROMA[ChromaDB Interface]
            VECTOR[Vector Store Manager]
            SEARCH[Semantic Search]
        end
        
        subgraph "Document Processing"
            PDF[PDF Loader]
            CHUNK[Text Splitter]
            ICD[ICD Code Processor]
        end
        
        subgraph "AI Integration"
            OPENAI[OpenAI Client]
            CHAIN[LangChain RAG]
            PROMPT[Prompt Templates]
        end
        
        subgraph "Caching Layer"
            REDIS_AI[Redis Cache]
            BINARY[Binary Cache]
        end
    end
    
    AI_APP --> NLP
    AI_APP --> VECTOR
    AI_APP --> PDF
    AI_APP --> OPENAI
    AI_APP --> REDIS_AI
```

### 3. WebSocket Communication Layer

```mermaid
graph TB
    subgraph "WebSocket Server"
        WS_SERVER[WebSocket Server :40510]
        
        subgraph "Connection Management"
            CONN_MAP[User Connections Map]
            SESSION[Session State Manager]
            AUTH_WS[JWT Authentication]
        end
        
        subgraph "Message Processing"
            MSG_HANDLER[Message Handler]
            QUERY_PROC[Query Processor]
            RESP_HANDLER[Response Handler]
        end
        
        subgraph "Integration"
            NLP_INT[NLP Service Integration]
            DB_LOG[Database Logging]
            ERROR_HAND[Error Handling]
        end
    end
    
    WS_SERVER --> CONN_MAP
    WS_SERVER --> MSG_HANDLER
    MSG_HANDLER --> NLP_INT
    NLP_INT --> DB_LOG
```

---

## Data Flow Architecture

### 1. Document Processing Flow

```mermaid
sequenceDiagram
    participant Client
    participant NodeAPI
    participant PythonAI
    participant MySQL
    participant ChromaDB
    participant Redis
    participant OpenAI
    
    Client->>NodeAPI: Upload Document
    NodeAPI->>MySQL: Store Document Metadata
    NodeAPI->>AIService: Process Document Request
    AIService->>AIService: Extract PDF Content
    AIService->>AIService: Split into Chunks
    AIService->>OpenAI: Generate Embeddings
    OpenAI-->>AIService: Return Embeddings
    AIService->>ChromaDB: Store Vector Embeddings
    AIService->>MySQL: Update Processing Status
    AIService->>Redis: Cache Vector Store
    AIService-->>NodeAPI: Processing Complete
    NodeAPI-->>Client: Success Response
```

### 2. Real-time Chat Flow

```mermaid
sequenceDiagram
    participant Client
    participant WebSocket
    participant NodeAPI
    participant PythonAI
    participant ChromaDB
    participant OpenAI
    participant MySQL
    
    Client->>WebSocket: Connect with JWT
    WebSocket->>NodeAPI: Validate Token
    NodeAPI->>MySQL: Verify User & Token
    MySQL-->>NodeAPI: User Validated
    NodeAPI-->>WebSocket: Authentication Success
    
    Client->>WebSocket: Send Query
    WebSocket->>AIService: Process Question
    AIService->>ChromaDB: Semantic Search
    ChromaDB-->>AIService: Relevant Documents
    AIService->>OpenAI: Generate Answer
    OpenAI-->>AIService: AI Response
    AIService-->>WebSocket: Formatted Answer
    WebSocket->>MySQL: Log Interaction
    WebSocket-->>Client: Send Response
```

### 3. User Authentication Flow

```mermaid
sequenceDiagram
    participant Client
    participant NodeAPI
    participant MySQL
    participant JWT
    participant Email
    
    Client->>NodeAPI: Login Request
    NodeAPI->>MySQL: Validate Credentials
    MySQL-->>NodeAPI: User Data
    NodeAPI->>JWT: Generate Tokens
    JWT-->>NodeAPI: Access & Refresh Tokens
    NodeAPI->>MySQL: Store Tokens
    NodeAPI-->>Client: Authentication Response
    
    Note over Client,NodeAPI: For subsequent requests
    Client->>NodeAPI: API Request + Token
    NodeAPI->>JWT: Verify Token
    NodeAPI->>MySQL: Check Token Validity
    MySQL-->>NodeAPI: Token Status
    NodeAPI-->>Client: Authorized Response
```

---

## Technology Stack

### Backend Services
```yaml
Primary API Server:
  Architecture: RESTful API
  Pattern: MVC (Model-View-Controller)
  Features: Middleware Pipeline, Route Handling
  
AI Processing Service:
  Architecture: Microservice
  Pattern: Service-Oriented Architecture
  Features: NLP Processing, Vector Operations
  
Process Manager:
  Features: Clustering, Auto-restart, Monitoring
  Pattern: Process Orchestration
```

### Data Storage
```yaml
Primary Database:
  Type: MySQL 2.18.1 / MySQL2 3.2.0
  Features: Connection Pooling, Retry Logic
  
Vector Database:
  Type: ChromaDB
  Purpose: Semantic Search, Embeddings Storage
  
Cache Layer:
  Type: Redis
  Purpose: Session Cache, Vector Store Cache
  
File Storage:
  Type: Local File System
  Purpose: Document Storage, Uploads
```

### AI/ML Stack
```yaml
NLP Processing:
  - Natural Language Understanding
  - Text Preprocessing & Tokenization
  - RAG (Retrieval Augmented Generation)
  - External AI API Integration
  
Vector Operations:
  - Embedding Generation
  - Vector Database Storage
  - Semantic Search & Retrieval
  
Document Processing:
  - Text Chunking & Splitting
  - PDF Content Extraction
  - Medical Code Processing
```

### Communication
```yaml
Real-time:
  Protocol: WebSocket (ws 8.13.0)
  Library: Socket.IO 4.8.1
  
HTTP API:
  Protocol: REST
  Security: JWT Authentication
  
External APIs:
  - OpenAI API
  - Email Services (Nodemailer)
```

---

## Database Architecture

### Entity Relationship Overview

```mermaid
erDiagram
    HOSPITALS ||--o{ USERS : manages
    HOSPITALS ||--o{ APP_USERS : serves
    HOSPITALS ||--o{ DOCUMENTS : owns
    USERS ||--o{ DOCUMENTS : uploads
    APP_USERS ||--o{ INTERACTION_LOGS : creates
    DOCUMENTS ||--o{ DOCUMENT_CHUNKS : contains
    
    HOSPITALS {
        int id PK
        string hospital_code UK
        string name
        string address
        timestamp created_at
        timestamp updated_at
    }
    
    USERS {
        int id PK
        int hospital_id FK
        string email UK
        string password_hash
        string role
        timestamp created_at
    }
    
    APP_USERS {
        int id PK
        string hospital_code FK
        string access_token
        string refresh_token
        timestamp token_expires_at
        timestamp created_at
    }
    
    DOCUMENTS {
        int id PK
        int hospital_id FK
        int uploaded_by FK
        string filename
        string file_path
        string status
        timestamp created_at
    }
    
    INTERACTION_LOGS {
        int id PK
        string session_id
        string hospital_code
        text query
        text response
        int app_user_id FK
        timestamp created_at
    }
```

### Database Configuration
```yaml
Connection Pool:
  Max Connections: 10
  Queue Limit: 0
  Connection Timeout: 10s
  Idle Timeout: 60s
  Keep Alive: Enabled
  
Retry Logic:
  Max Retries: 3
  Backoff Strategy: Exponential
  Retry Conditions:
    - PROTOCOL_CONNECTION_LOST
    - ECONNRESET
    - PROTOCOL_ENQUEUE_AFTER_FATAL_ERROR
```

---

## Security Architecture

### Authentication & Authorization

```mermaid
graph TB
    subgraph "Authentication Layer"
        JWT[JWT Token System]
        ACCESS[Access Tokens]
        REFRESH[Refresh Tokens]
        EXPIRE[Token Expiration]
    end
    
    subgraph "Authorization Layer"
        RBAC[Role-Based Access Control]
        HOSP[Hospital-Based Isolation]
        PERM[Permission Validation]
    end
    
    subgraph "Security Middleware"
        HELMET[Helmet.js Security Headers]
        CORS_SEC[CORS Configuration]
        RATE_LIM[Rate Limiting]
        INPUT_VAL[Input Validation]
    end
    
    subgraph "Data Security"
        ENCRYPT[Password Encryption - bcrypt]
        HASH[Data Hashing]
        SANITIZE[Input Sanitization]
    end
    
    JWT --> RBAC
    RBAC --> HELMET
    HELMET --> ENCRYPT
```

### Security Features
```yaml
Authentication:
  Method: JWT (JSON Web Tokens)
  Token Types: Access & Refresh
  Expiration: Configurable
  Storage: Database + Redis Cache
  
Password Security:
  Hashing: bcrypt (5.1.1)
  Salt Rounds: Configurable
  Validation: Joi Schema Validation
  
Request Security:
  Headers: Helmet.js
  CORS: Configurable Origins
  Rate Limiting: Express Rate Limit
  Input Validation: Joi Schemas
  
Data Isolation:
  Multi-tenancy: Hospital-based
  Vector Store: Hospital-specific collections
  Database: Hospital ID filtering
```

---

## Deployment Architecture

### Process Management

```mermaid
graph TB
    subgraph "PM2 Process Manager"
        PM2[PM2 Master Process]
        
        subgraph "Node.js Cluster"
            NODE1[Node.js Instance 1]
            NODE2[Node.js Instance 2]
            NODE3[Node.js Instance N]
        end
        
        subgraph "Python Services"
            PYTHON1[Flask AI Service 1]
            PYTHON2[Flask AI Service 2]
        end
        
        subgraph "WebSocket Services"
            WS1[WebSocket Server 1]
            WS2[WebSocket Server 2]
        end
    end
    
    subgraph "Monitoring"
        LOGS[Log Aggregation]
        METRICS[Performance Metrics]
        HEALTH[Health Checks]
    end
    
    PM2 --> NODE1
    PM2 --> NODE2
    PM2 --> NODE3
    PM2 --> PYTHON1
    PM2 --> PYTHON2
    PM2 --> WS1
    PM2 --> WS2
    PM2 --> LOGS
```

### Service Configuration
```yaml
Node.js API:
  Port: 3000 (configurable)
  Instances: Auto (CPU cores)
  Restart Policy: Always
  Max Memory: 1GB
  
Python AI Service:
  Port: 5000
  Instances: 1-2
  Restart Policy: Always
  Max Memory: 2GB
  
WebSocket Server:
  Port: 40510
  Instances: 1-2
  Protocol: ws://
  Max Connections: 1000
  
Logging:
  Levels: INFO, ERROR, DEBUG
  Rotation: Daily
  Retention: 30 days
  Format: JSON structured
```

### Environment Configuration
```yaml
Development:
  Debug: Enabled
  Hot Reload: Nodemon
  Database: Local MySQL
  Cache: Local Redis
  
Production:
  Debug: Disabled
  Process Manager: PM2
  Database: Production MySQL
  Cache: Redis Cluster
  SSL: Required
  Monitoring: Enabled
```

---

## Performance Considerations

### Optimization Strategies
```yaml
Database:
  - Connection pooling with retry logic
  - Query optimization with indexes
  - Hospital-based data partitioning
  
Caching:
  - Redis for vector store caching
  - Session state caching
  - Query result caching (1 hour TTL)
  
Vector Operations:
  - Batch processing for embeddings
  - Chunked document processing
  - Async vector store operations
  
WebSocket:
  - Connection pooling
  - Message queuing
  - Graceful connection cleanup
```

### Scalability Features
```yaml
Horizontal Scaling:
  - Stateless API design
  - Load balancer ready
  - Database connection pooling
  
Vertical Scaling:
  - Memory-efficient processing
  - Async operations
  - Resource monitoring
  
Data Scaling:
  - Hospital-based partitioning
  - Vector store sharding
  - Incremental processing
```

---

*This documentation provides a comprehensive overview of the SpurrinAI backend architecture. For implementation details, refer to the source code and individual component documentation.*
