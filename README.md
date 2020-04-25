# EndpointSecurity API Documentation

This repository contains documentation and diagrams about Apple's EndpointSecurity API framework, which provides system event monitoring and security capabilities on macOS.

## Diagrams

### EndpointSecurity Event Types

```mermaid
flowchart LR
    E[System Event] --> ES[EndpointSecurity]

    ES --> N[NOTIFY Event]
    ES --> A[AUTH Event]

    subgraph "NOTIFY Events"
        N --> NF[Asynchronous]
        NF --> NF1[Operation continues immediately]
        NF --> NF2[Contains result field]
        NF --> NF3[Always delivered unless muted]
    end

    subgraph "AUTH Events"
        A --> AF[Synchronous]
        AF --> AF1[Operation blocked until response]
        AF --> AF2[Has deadline field]
        AF --> AF3[Only delivered if no cached result]
        AF --> AF4[Requires client response]
        AF --> AF5[Can be cached]
    end

    style N fill:#aaf,stroke:#33a,stroke-width:2px
    style A fill:#faa,stroke:#a33,stroke-width:2px
```

This diagram illustrates the two main types of EndpointSecurity events: NOTIFY and AUTH events, and their key differences.

### EndpointSecurity Early Boot Process

```mermaid
sequenceDiagram
    participant User
    participant System as macOS System
    participant SysExt as System Extension
    participant ES as EndpointSecurity Framework
    participant Apps as Third-Party Apps

    System->>System: Boot Start
    System->>SysExt: Load Early Boot Extensions

    SysExt->>ES: Initialize (es_new_client)
    ES-->>SysExt: Client Handle

    SysExt->>ES: First Subscribe Call
    ES-->>SysExt: Subscription Result

    alt All Early Boot Extensions Ready
        System->>Apps: Allow Third-Party Apps to Execute
    else Early Boot Deadline Expires
        System->>System: Deadline Timer
        System->>Apps: Force Allow Third-Party Apps
    end

    User->>System: Try to Use Apps

    Note over User,Apps: System Extensions installed in /Applications<br>Protected by System Integrity Protection<br>Protected from root user unload
```

This sequence diagram shows how EndpointSecurity extensions are loaded during the early boot process and how they interact with the system.

### EndpointSecurity Client Flow

```mermaid
sequenceDiagram
    participant App as ES Client Application
    participant ES as EndpointSecurity Framework
    participant K as Kernel

    App->>ES: es_new_client()
    ES-->>App: client handle + event handler block
    App->>ES: es_subscribe(events)
    ES-->>App: subscription result

    App->>App: dispatch_main()

    K->>ES: Event occurs
    ES->>ES: Enrich event data

    alt is AUTH event
        ES->>App: Deliver AUTH message
        App->>App: Process message
        App->>ES: es_respond_auth_result()
        ES->>K: Apply response
    else is NOTIFY event
        ES->>App: Deliver NOTIFY message
        App->>App: Process message
        ES->>K: Operation already continued
    end

    opt Async processing needed
        App->>App: es_copy_message()
        App->>App: Queue work on async queue
        App->>App: Process later
        App->>ES: es_respond_auth_result() if needed
        App->>App: es_free_message()
    end
```

This diagram demonstrates the typical flow of an EndpointSecurity client application, from initialization to event handling.

### EndpointSecurity Caching System

```mermaid
flowchart TD
    E[Event Occurs] --> CE{Cached Entry Exists?}

    CE -->|Yes| CMP{Compare with Cache}
    CE -->|No| D[Deliver to Clients]

    CMP -->|Match| AR[Apply Cached Result]
    CMP -->|No Match| D

    D --> MULTI{Multiple Clients?}

    MULTI -->|Yes| C1[Client 1 Response]
    MULTI -->|Yes| C2[Client 2 Response]
    MULTI -->|No| SR[Single Response]

    C1 --> COMB[Combine Responses]
    C2 --> COMB

    COMB --> MR[Most Restrictive Rule Applied]
    SR --> FR[Final Response]
    MR --> FR

    FR -->|cache=true| ADDCACHE[Add to Cache]
    FR -->|cache=false| NOCACHE[Skip Cache]

    ADDCACHE --> RET[Return Result to Kernel]
    NOCACHE --> RET
    AR --> RET

    subgraph "Cache Invalidation"
        INV[File Modified/Deleted]
        CONNECT[New Client Connects]
        DISCONNECT[Client Disconnects]
        CLEAR[es_clear_cache() Called]

        INV --> INVC[Invalidate Cache Entry]
        CONNECT --> INVC
        DISCONNECT --> INVC
        CLEAR --> INVC
    end

    style ADDCACHE fill:#9cf,stroke:#333,stroke-width:2px
    style INVC fill:#f99,stroke:#333,stroke-width:2px
```

This flowchart explains how the EndpointSecurity caching system works to improve performance and reduce redundant event processing.

### EndpointSecurity Architecture

```mermaid
flowchart TD
    K[Kernel Event] --> ES[EndpointSecurity Subsystem]
    ES --> |"Intercept & Enrich"| MSG[Message Creation]
    MSG --> |"Wrapped in envelope"| Q[Message Queue]
    Q --> |"Simultaneously deliver to subscribed clients"| C1[ES Client 1]
    Q --> |"Simultaneously deliver to subscribed clients"| C2[ES Client 2]
    Q --> |"Simultaneously deliver to subscribed clients"| C3[ES Client 3]

    C1 --> |"if AUTH event"| R1[Response required]
    C2 --> |"if AUTH event"| R2[Response required]
    C3 --> |"if AUTH event"| R3[Response required]

    R1 --> RESP[Combined Response]
    R2 --> RESP
    R3 --> RESP

    RESP --> |"ALLOW/DENY"| K2[Kernel Operation Continues]

    subgraph "Cache System"
        RESP --> CACHE[Global Cache]
        CACHE --> |"Future similar events"| Q
    end

    style ES fill:#f96,stroke:#333,stroke-width:2px
    style CACHE fill:#9cf,stroke:#333,stroke-width:2px
```

This diagram provides an overview of the EndpointSecurity subsystem architecture and how events flow through the system.

## Notes

- The diagrams are created using Mermaid syntax and can be viewed directly in GitHub.
- For an interactive experience with these diagrams in Obsidian, clone this repository and open it as an Obsidian vault.
