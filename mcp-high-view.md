# MCP Gateway High-Level Architecture

```mermaid
graph TB
    subgraph "Entry Point"
        WS[Web Server<br/>HTTP/REST API]
    end

    subgraph "Message Bus Layer 1"
        MB1[Message Bus<br/>Task Queue]
    end

    subgraph "Pre-Processing Layer"
        PC1[Plugin Container 1<br/>Pre-Processing]
        PC2[Plugin Container 2<br/>Pre-Processing]
        PC3[Plugin Container N<br/>Pre-Processing]
    end

    subgraph "Message Bus Layer 2"
        MB2[Message Bus<br/>Results Queue]
    end

    subgraph "MCP Communication Layer"
        MCP[MCP Server Connector<br/>Upstream Communication]
    end

    subgraph "Message Bus Layer 3"
        MB3[Message Bus<br/>Response Queue]
    end

    subgraph "Post-Processing Layer"
        PP1[Plugin Container 1<br/>Post-Processing]
        PP2[Plugin Container 2<br/>Post-Processing]
        PP3[Plugin Container N<br/>Post-Processing]
    end

    subgraph "Message Bus Layer 4"
        MB4[Message Bus<br/>Final Results Queue]
    end

    subgraph "Upstream Services"
        US1[MCP Server 1]
        US2[MCP Server 2]
        US3[MCP Server N]
    end

    %% Request Flow
    WS -->|1. Publish Request| MB1
    MB1 -->|2. Consume Tasks| PC1
    MB1 -->|2. Consume Tasks| PC2
    MB1 -->|2. Consume Tasks| PC3
    
    PC1 -->|3. Publish Results| MB2
    PC2 -->|3. Publish Results| MB2
    PC3 -->|3. Publish Results| MB2
    
    MB2 -->|4. Consume Results| MCP
    
    MCP -->|5. Send Requests| US1
    MCP -->|5. Send Requests| US2
    MCP -->|5. Send Requests| US3
    
    US1 -->|6. Return Responses| MCP
    US2 -->|6. Return Responses| MCP
    US3 -->|6. Return Responses| MCP
    
    MCP -->|7. Publish Responses| MB3
    
    MB3 -->|8. Consume Responses| PP1
    MB3 -->|8. Consume Responses| PP2
    MB3 -->|8. Consume Responses| PP3
    
    PP1 -->|9. Publish Final Results| MB4
    PP2 -->|9. Publish Final Results| MB4
    PP3 -->|9. Publish Final Results| MB4
    
    MB4 -->|10. Consume & Return| WS

    %% Styling
    classDef webServer fill:#4CAF50,stroke:#2E7D32,stroke-width:3px,color:#fff
    classDef messageBus fill:#2196F3,stroke:#1565C0,stroke-width:2px,color:#fff
    classDef plugin fill:#FF9800,stroke:#E65100,stroke-width:2px,color:#fff
    classDef mcp fill:#9C27B0,stroke:#6A1B9A,stroke-width:3px,color:#fff
    classDef upstream fill:#607D8B,stroke:#37474F,stroke-width:2px,color:#fff

    class WS webServer
    class MB1,MB2,MB3,MB4 messageBus
    class PC1,PC2,PC3,PP1,PP2,PP3 plugin
    class MCP mcp
    class US1,US2,US3 upstream
```

## Architecture Components

### 1. Web Server (Entry Point)
- **Role**: Receives incoming HTTP/REST API requests
- **Actions**: 
  - Validates requests
  - Publishes tasks to Message Bus 1
  - Receives final results from Message Bus 4
  - Returns responses to clients

### 2. Message Bus Layer 1 (Task Queue)
- **Role**: Distributes incoming tasks to pre-processing plugins
- **Pattern**: Pub/Sub or Work Queue
- **Consumers**: Plugin Containers (Pre-Processing)

### 3. Pre-Processing Plugin Containers
- **Role**: Process and transform incoming requests
- **Scalability**: Multiple instances for horizontal scaling
- **Actions**:
  - Consume tasks from Message Bus 1
  - Apply pre-processing logic (validation, transformation, enrichment)
  - Publish results to Message Bus 2

### 4. Message Bus Layer 2 (Results Queue)
- **Role**: Aggregates pre-processed results
- **Consumer**: MCP Server Connector

### 5. MCP Server Connector
- **Role**: Communicates with upstream MCP servers
- **Actions**:
  - Consumes pre-processed tasks from Message Bus 2
  - Sends requests to upstream MCP servers
  - Receives responses from MCP servers
  - Publishes responses to Message Bus 3

### 6. Upstream MCP Servers
- **Role**: External MCP protocol servers
- **Examples**: Tool servers, resource servers, prompt servers

### 7. Message Bus Layer 3 (Response Queue)
- **Role**: Distributes MCP server responses to post-processing plugins
- **Consumers**: Plugin Containers (Post-Processing)

### 8. Post-Processing Plugin Containers
- **Role**: Process and transform MCP server responses
- **Scalability**: Multiple instances for horizontal scaling
- **Actions**:
  - Consume responses from Message Bus 3
  - Apply post-processing logic (filtering, formatting, aggregation)
  - Publish final results to Message Bus 4

### 9. Message Bus Layer 4 (Final Results Queue)
- **Role**: Delivers final processed results back to web server
- **Consumer**: Web Server

## Data Flow Summary

1. **Client → Web Server**: HTTP request
2. **Web Server → MB1**: Publish task
3. **MB1 → Pre-Processing Plugins**: Distribute tasks
4. **Pre-Processing Plugins → MB2**: Publish processed tasks
5. **MB2 → MCP Connector**: Consume processed tasks
6. **MCP Connector → Upstream MCP Servers**: Send MCP requests
7. **Upstream MCP Servers → MCP Connector**: Return MCP responses
8. **MCP Connector → MB3**: Publish responses
9. **MB3 → Post-Processing Plugins**: Distribute responses
10. **Post-Processing Plugins → MB4**: Publish final results
11. **MB4 → Web Server**: Consume final results
12. **Web Server → Client**: HTTP response

## Scalability & Resilience

- **Horizontal Scaling**: Plugin containers can scale independently
- **Decoupling**: Message buses decouple components
- **Fault Tolerance**: Failed plugins don't affect other components
- **Load Distribution**: Message buses distribute load across plugin instances
- **Async Processing**: Non-blocking message-based communication
