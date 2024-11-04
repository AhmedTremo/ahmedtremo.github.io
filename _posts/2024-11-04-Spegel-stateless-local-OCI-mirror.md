---
title: Spegel: Stateless Local OCI Mirror
date: 2024-11-04
categories: [OCI, Mirror, Spegel]
tags: [OCI, Mirror, Spegel]
author: tremo
---

[Spegel](https://github.com/spegel-org/spegel) describes itself as a stateless, cluster-local OCI registry mirror. In this post, we’ll decipher what that means and explore why Spegel might be valuable for your Kubernetes (K8s) setup.

## What is Spegel?

1. **Stateless**: Spegel doesn’t store image manifests or layers itself; it relies on containerd’s store on the node.
2. **Local**: It operates within your local K8s cluster.
3. **OCI Registry**: It’s compliant with the OCI distribution specification.
4. **Mirror**: It mirrors images from a remote registry to your local cluster. (_Fun fact: "Spegel" means mirror in Swedish._)

## Why Use Spegel?

1. **Reduce Bandwidth Usage** to remote registries.
2. **Faster Image Pulls** if images are cached within the K8s cluster.
3. **Fault Tolerance** if the remote registry is unavailable.

## How Spegel Works

This blog focuses on visual diagrams that illustrate Spegel's inner workings, making it easier to map component connections and debug issues.

### Spegel Overview

Nodes in the cluster that need an image will first check if it’s available locally.

- If **available**, they pull it directly from other nodes using the P2P network.
- If **not available**, the system falls back to the remote registry.

![Spegel Flow Diagram](/assets/img/posts/2024-11-04-Spegel-stateless-local-OCI-mirror/overview_gif.gif)

## Spegel: Visual Architecture Guide

Let’s dive into Spegel’s architecture through a series of diagrams:

### 1. High-Level Cluster Architecture

This diagram shows how Spegel pods form a P2P network within the cluster. Each Spegel pod interacts with containerd and, if necessary, falls back to the external registry.

```mermaid
graph TB
    subgraph "External"
        ER["External Registry"]
    end

    subgraph "Kubernetes Cluster"
        subgraph "Node 1"
            SP1["Spegel Pod"]
            CD1["Containerd"]
            SP1 <-->|interacts| CD1
            CD1 -->|fallback| ER
        end
        
        subgraph "Node 2"
            SP2["Spegel Pod"]
            CD2["Containerd"]
            SP2 <-->|interacts| CD2
            CD2 -->|fallback| ER
        end
        
        subgraph "Node 3"
            SP3["Spegel Pod"]
            CD3["Containerd"]
            SP3 <-->|interacts| CD3
            CD3 -->|fallback| ER
        end

        SP1 <-->|P2P Network| SP2
        SP2 <-->|P2P Network| SP3
        SP3 <-->|P2P Network| SP1
    end
```

### 2. Pod Component Architecture

Displays the components within a Spegel pod, including registry services, P2P components, and state management.

```mermaid
graph TB
    subgraph "Spegel Pod"
        subgraph "Registry Service"
            RS[HTTP Server /v2/]
            RH[Request Handler]
            RS --> RH
        end

        subgraph "P2P Components"
            P2P[P2P Router]
            DHT[DHT Provider]
            BS[Bootstrapper]
            P2P --> DHT
            BS --> P2P
        end

        subgraph "State Management"
            ST[State Tracker]
            MT[Metrics]
            ST --> MT
        end

        CD[Containerd Client]
        
        RH --> P2P
        ST --> P2P
        CD --> ST
    end

    subgraph "Node Components"
        CDD[Containerd Daemon]
        CS[Content Store]
        CDD --> CS
    end

    CD --> CDD
```

### 3. Image Pull Flow

This sequence shows how an image pull request is handled, covering both peer pulls and fallback to external registry.

```mermaid
sequenceDiagram
    participant CD as Containerd
    participant SR as Spegel Registry
    participant P2P as P2P Router
    participant PR as Peer Registry
    participant ER as External Registry

    Note over SR,P2P: 20ms default resolve timeout
    Note over SR,P2P: 3 default resolve retries

    CD->>SR: GET /v2/{name}/manifests/{ref}
    SR->>P2P: Resolve(key, allowSelf, retries)
    
    alt Peer Found
        P2P-->>SR: Return Peer Address
        SR->>PR: Request Content
        PR-->>SR: Stream Content
        SR-->>CD: Return Content
        CD->>CS: Store Content
    else No Peers Available (within 20ms)
        SR-->>CD: 404 Not Found
        CD->>ER: Request from External
        ER-->>CD: Return Content
        CD->>CS: Store Content
    end
```

### 4. P2P Network Formation

Illustrates how nodes discover each other and form the P2P network through leader election and peer sharing.

```mermaid
sequenceDiagram
    participant N1 as Node 1
    participant N2 as Node 2
    participant N3 as Node 3
    participant LE as Leader Election
    
    Note over N1,LE: 10s lease duration
    Note over N1,LE: 5s renew deadline
    Note over N1,LE: 2s retry period

    N1->>LE: Participate in Election
    N2->>LE: Participate in Election
    N3->>LE: Participate in Election
    LE->>N1: Elected Leader
    N2->>N1: Discover Leader
    N3->>N1: Discover Leader
    N1->>N2: Share Peer List
    N1->>N3: Share Peer List
    N2->>N3: Establish P2P Connection
    
    Note over N1,N3: P2P Network Formed
```

### 5. State Management and Content Advertisement

Depicts how content availability is maintained and advertised across the P2P network.

```mermaid
sequenceDiagram
    participant ST as State Tracker
    participant CD as Containerd
    participant P2P as P2P Router
    participant DHT as DHT Network
    participant MT as Metrics

    Note over ST,DHT: Content TTL: 10 minutes
    Note over ST,DHT: Refresh: Every 9 minutes

    loop Every 9 minutes
        ST->>CD: List Images
        CD-->>ST: Image List
        
        loop For each image
            ST->>P2P: Advertise(image_keys)
            P2P->>DHT: Provide(keys)
        end
        
        ST->>MT: Update Metrics
    end

    CD-->>ST: Image Event (Create/Update/Delete)
    ST->>P2P: Update Advertisement
    ST->>MT: Update Metrics
```

### 6. Content Resolution Process

Shows content location and retrieval, including peer selection and retry mechanisms.

```mermaid
sequenceDiagram
    participant SR as Spegel Registry
    participant P2P as P2P Router
    participant DHT as DHT Network
    participant PR1 as Peer 1
    participant PR2 as Peer 2

    SR->>P2P: Resolve(content_key)
    P2P->>DHT: FindProviders(key)
    
    par Parallel Resolution
        DHT-->>P2P: Found Peer 1
        DHT-->>P2P: Found Peer 2
    end

    P2P->>SR: Return First Available Peer
    
    Note over SR,PR2: Default 20ms timeout
    Note over SR,PR2: 3 retry attempts
    
    alt Try Peer 1
        SR->>PR1: Request Content
        PR1-->>SR: Stream Content
    else Peer 1 Fails
        SR->>PR2: Request Content
        PR2-->>SR: Stream Content
    end
```

### 7. Data Flow Paths

Describes content and control flow within the system, including peer transfers and fallback.

```mermaid
graph LR
    subgraph "Content Paths"
        CD[Containerd]
        SP[Spegel]
        P[Peers]
        ER[External Registry]
        CS[Content Store]
        
        CD -->|Request| SP
        SP -->|Check| P
        P -->|Content| SP
        SP -->|Return| CD
        CD -->|Store| CS

        SP -->|404| CD
        CD -->|Fallback| ER
    end

    subgraph "P2P Operations"
        P2P[P2P Network]
        DHT[DHT]
        ST[State Tracker]
        
        P2P -->|Advertise| DHT
        DHT -->|Discover| P2P
        ST -->|Update| P2P
    end
```

### 8. Failure Handling

Demonstrates failure handling scenarios within the system.

```mermaid
sequenceDiagram
    participant CD as Containerd
    participant SR as Spegel Registry
    participant P2P as P2P Router
    participant PR as Peer
    participant ER as External Registry

    Note over SR,ER: Failure Scenarios

    alt Peer Not Found
        CD->>SR: Request Content
        SR->>P2P: Resolve(key)
        P2P--xSR: No Peers Available
        SR-->>CD: 404 Not Found
        CD->>ER: Fallback Request
    end

    alt Peer Connection Failed
        SR->>PR: Request Content
        PR--xSR: Connection Failed
        SR->>P2P: Resolve(key) Retry
        P2P-->>SR: Alternative Peer
    end

    alt Content Corrupted
        SR->>PR: Request Content
        PR-->>SR: Stream Content
        SR--xCD: Verification Failed
        CD->>ER: Fallback Request
    end
```

### 9. Metrics Collection

Shows how metrics are collected and organized across the system components.

```mermaid
graph TB
    subgraph "Metrics Sources"
        RQ[Registry Requests]
        P2P[P2P Operations]
        ST[State Changes]
    end

    subgraph "Metric Types"
        CT[Counters]
        HT[Histograms]
        GT[Gauges]
    end

    subgraph "Prometheus Metrics"
        MR[mirror_requests_total]
        RD[resolve_duration_seconds]
        AI[advertised_images]
        AK[advertised_keys]
        RL[request_latency]
        IF[requests_inflight]
    end

    RQ --> CT
    RQ --> HT
    P2P --> HT
    P2P --> GT
    ST --> GT

    CT --> MR
    HT --> RD
    HT --> RL
    GT --> AI
    GT --> AK
    GT --> IF
```