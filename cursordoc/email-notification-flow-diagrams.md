# Email Notification System - Flow Diagrams

## 1. Overall System Architecture

```mermaid
graph TB
    A[Policy Revision/Deviation Created] --> B[Approval Adapter]
    B --> C[Create Approval Instance]
    C --> D[Notification Service]
    D --> E[Email Queue Service]
    E --> F[Redis Queue]
    F --> G[Email Queue Processor]
    G --> H[Email Service]
    H --> I[SMTP Server]
    I --> J{Success?}
    J -->|Yes| K[Mark Success]
    J -->|No| L{Error Type?}
    L -->|Retryable| M[Schedule Retry]
    L -->|Permanent| N[Move to Failed Queue]
    M --> F
```

## 2. Approval Instance Creation Flow

```mermaid
sequenceDiagram
    participant User
    participant Adapter
    participant Notification
    participant Queue
    participant Redis
    participant Processor
    participant SMTP

    User->>Adapter: Create Revision/Deviation
    Adapter->>Adapter: Create Approval Instance
    Adapter->>Notification: notify_*_created()
    Notification->>Notification: Get Approver Info
    Notification->>Notification: Generate Email Content
    Notification->>Queue: enqueue_email(priority=True)
    Queue->>Redis: LPUSH priority queue
    Note over Redis: Email in Priority Queue
    Processor->>Redis: RPOP priority queue
    Redis->>Processor: Email Data
    Processor->>SMTP: Send Email
    SMTP-->>Processor: Success/Failure
    Processor->>Queue: Mark Success/Failed
```

## 3. Approval Decision Flow

```mermaid
flowchart TD
    A[Approver Makes Decision] --> B{Decision Type}
    B -->|Approve| C{Has Next Approver?}
    B -->|Reject| D[Workflow Failed]
    
    C -->|Yes| E[Notify Next Approver]
    C -->|No| F[Workflow Completed]
    
    D --> G[Notify Creator - Rejected]
    F --> H{Decision Type}
    H -->|Approve| I[Notify Creator - Approved]
    H -->|Reject| G
    
    E --> J[Email Queue Priority]
    I --> K[Email Queue Normal]
    G --> K
```

## 4. Email Queue Processing Flow

```mermaid
stateDiagram-v2
    [*] --> Pending: Email Enqueued
    Pending --> Priority: High Priority
    Priority --> Processing: Dequeued
    Pending --> Processing: Dequeued
    Processing --> Success: Send Success
    Processing --> Retry: Retryable Error
    Processing --> Failed: Permanent Error
    Retry --> Priority: Re-enqueue
    Retry --> Failed: Max Retries
    Success --> [*]
    Failed --> [*]
    
    note right of Retry
        Exponential Backoff:
        1min → 5min → 15min
        → 1h → 6h → 24h
    end note
```

## 5. Retry Mechanism Flow

```mermaid
flowchart LR
    A[Email Send Failed] --> B{Retry Count < 6?}
    B -->|Yes| C[Calculate Delay]
    B -->|No| D[Move to Failed Queue]
    
    C --> E{Retry Count}
    E -->|1| F[Delay: 1 min]
    E -->|2| G[Delay: 5 min]
    E -->|3| H[Delay: 15 min]
    E -->|4| I[Delay: 1 hour]
    E -->|5| J[Delay: 6 hours]
    E -->|6| K[Delay: 24 hours]
    
    F --> L[Store Retry Info]
    G --> L
    H --> L
    I --> L
    J --> L
    K --> L
    
    L --> M[Re-enqueue Priority]
    M --> N[Wait for Retry Time]
    N --> O[Process Again]
    O --> P{Success?}
    P -->|Yes| Q[Mark Success]
    P -->|No| A
```

## 6. Background Task Processing

```mermaid
sequenceDiagram
    participant App
    participant BackgroundService
    participant Redis
    participant Processor
    participant EmailService
    participant SMTP

    App->>BackgroundService: Startup Event
    BackgroundService->>Redis: Check Connection
    Redis-->>BackgroundService: Connected
    BackgroundService->>Processor: Initialize
    BackgroundService->>BackgroundService: Schedule Task (5s interval)
    
    loop Every 5 seconds
        BackgroundService->>Processor: process_batch()
        Processor->>Redis: Cleanup Stale Processing
        Processor->>Redis: RPOP Priority Queue
        Redis-->>Processor: Email Data
        Processor->>Redis: ZADD Processing Queue
        Processor->>EmailService: send_email()
        EmailService->>SMTP: Connect & Send
        SMTP-->>EmailService: Result
        EmailService-->>Processor: Success/Error
        Processor->>Redis: Mark Success/Failed
    end
```

## 7. Error Handling Flow

```mermaid
flowchart TD
    A[Email Send Attempt] --> B{Error Type}
    
    B -->|SMTPConnectError| C[Retryable]
    B -->|TimeoutError| C
    B -->|SMTPDataError| C
    B -->|SMTPServerDisconnected| C
    
    B -->|SMTPAuthenticationError| D[Permanent]
    B -->|SMTPRecipientsRefused| D
    B -->|SMTPSenderRefused| D
    
    C --> E{Retry Count < Max?}
    E -->|Yes| F[Schedule Retry]
    E -->|No| G[Move to Failed Queue]
    
    F --> H[Calculate Exponential Delay]
    H --> I[Store Retry Info]
    I --> J[Re-enqueue Priority]
    
    D --> G
    G --> K[Dead Letter Queue]
    K --> L[Log Error]
```

## 8. Component Interaction Diagram

```mermaid
graph LR
    subgraph "Approval Layer"
        A1[Policy Approval Adapter]
        A2[Deviation Approval Adapter]
    end
    
    subgraph "Notification Layer"
        N[Notification Service]
        T[Email Templates]
    end
    
    subgraph "Queue Layer"
        Q[Email Queue Service]
        R[(Redis)]
    end
    
    subgraph "Processing Layer"
        P[Email Queue Processor]
        E[Email Service]
    end
    
    subgraph "External"
        S[SMTP Server]
    end
    
    A1 --> N
    A2 --> N
    N --> T
    N --> Q
    Q --> R
    P --> R
    P --> E
    E --> S
    
    style R fill:#ff9999
    style S fill:#99ff99
```

## 9. Data Flow Diagram

```mermaid
flowchart TD
    A[Approval Event] --> B[Notification Service]
    B --> C[Generate Email Data]
    C --> D{Email Data Structure}
    
    D --> E["id: uuid<br/>to_email: string<br/>subject: string<br/>html_content: string<br/>metadata: object<br/>retry_count: int<br/>priority: bool"]
    
    E --> F[Redis Queue]
    F --> G[Email Queue Processor]
    G --> H[Email Service]
    H --> I[SMTP Server]
    
    I --> J{Result}
    J -->|Success| K[Update Stats]
    J -->|Failure| L[Update Retry Info]
    
    K --> M[Remove from Queue]
    L --> N{Retry?}
    N -->|Yes| F
    N -->|No| O[Failed Queue]
```

## 10. Complete Notification Lifecycle

```mermaid
stateDiagram-v2
    [*] --> Enqueued: Approval Event
    Enqueued --> PriorityQueue: High Priority
    Enqueued --> PendingQueue: Normal Priority
    
    PriorityQueue --> Processing: Dequeued
    PendingQueue --> Processing: Dequeued
    
    Processing --> Sending: Send Email
    Sending --> Success: Email Sent
    Sending --> RetryableError: Connection Error
    Sending --> PermanentError: Invalid Email
    
    RetryableError --> RetryScheduled: Count < Max
    RetryScheduled --> PriorityQueue: Re-enqueue
    RetryableError --> FailedQueue: Count >= Max
    
    PermanentError --> FailedQueue: No Retry
    
    Success --> [*]
    FailedQueue --> [*]
    
    note right of RetryScheduled
        Exponential Backoff
        Delays: 1m, 5m, 15m, 1h, 6h, 24h
    end note
```

