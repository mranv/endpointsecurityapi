# EndpointSecurity Framework Documentation

This repository contains comprehensive documentation and visualizations for Apple's EndpointSecurity Framework, which provides system event monitoring and security capabilities on macOS.

## Overview

The EndpointSecurity Framework (introduced in macOS Catalina) is designed as a modern replacement for:
- Kernel Extensions (KEXTs)
- Kauth KPI
- Unsupported Mac kernel framework
- OpenBSM audit trail

It allows security products to monitor and control system events from user space without kernel extensions, improving system stability and security.

## Core Concepts Visualized

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

### Message Structure

```mermaid
classDiagram
    class es_message_t {
        +Metadata
        +Process Information
        +Event Specific Data
    }

    class Metadata {
        +event_type
        +time
        +version
        +is_auth_or_notify
        +deadline (for AUTH)
        +sequence_number
    }

    class ProcessInfo {
        +executable_path
        +pid
        +uid
        +code_signing_info
        +audit_token
    }

    class EventSpecificData {
        +exec_data (for EXEC)
        +open_data (for OPEN)
        +signal_data (for SIGNAL)
        +etc...
    }

    es_message_t --> Metadata
    es_message_t --> ProcessInfo
    es_message_t --> EventSpecificData
```

### AUTH vs NOTIFY Events

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

### Client Initialization and Event Flow

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

### Caching and Response Mechanism

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

### System Extension and Early Boot Process

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

## Key Features

### Event Types
- **NOTIFY Events**: Informational events that tell you an operation is happening
- **AUTH Events**: Control events that allow you to approve or deny operations

### Implementation Benefits
- C library for performance and compatibility (callable from Swift, Objective-C, Rust)
- No kernel extension required
- Enhanced system stability and security
- Rich event stream (~100 event types)

### Security Enhancements
- System Extension provides System Integrity Protection
- Protection from root user tampering
- Early boot capability for security initialization before third-party apps

### Performance Considerations
- Caching mechanism to improve performance
- Message muting to reduce event volume
- Asynchronous processing recommended for AUTH events
- Response deadlines must be respected

## Getting Started

### Requirements
- Endpoint Security entitlement (restricted, requires approval)
- Full Disk Access permissions (for privacy)
- System Extension entitlement (if deployed as system extension)
- User approval for installation

### Basic Implementation Flow
1. Initialize client with `es_new_client()`
2. Set up event handler block
3. Subscribe to events with `es_subscribe()`
4. Process events in handler block
5. For AUTH events, respond before deadline expires

### Best Practices
- Avoid time-of-check time-of-use issues
- Use message muting to reduce event volume
- Use caching for performance, not policy
- Process messages asynchronously when possible
- Split subscriptions across multiple clients if needed

## Usage in Obsidian

The Mermaid diagrams in this README are compatible with Obsidian and can be viewed directly in your Obsidian vault. Place this README in your vault and the diagrams will render automatically.

For Obsidian usage, you can save these diagrams in separate files:
- `assets/mermid/es-architecture.mermaid`
- `assets/mermid/es-message-structure.mermaid`
- `assets/mermid/es-event-types.mermaid`
- `assets/mermid/es-client-flow.mermaid`
- `assets/mermid/es-caching.mermaid`
- `assets/mermid/es-early-boot.mermaid`

Then reference them in your Obsidian notes using:
```
![[assets/mermid/es-architecture.mermaid]]
```

## References

- [Apple Developer Documentation](https://developer.apple.com/documentation)
- [WWDC Session: Build an Endpoint Security app](https://developer.apple.com/videos/play/wwdc2020/)
- [System Extensions and DriverKit](https://developer.apple.com/videos/play/wwdc2019/)

## Notes

This documentation is based on the EndpointSecurity Framework as described in WWDC presentations and official documentation. The framework continues to evolve with new macOS releases, adding events and capabilities.
