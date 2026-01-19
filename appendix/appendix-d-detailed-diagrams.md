# Appendix D: Detailed Diagrams

## D.1 Capability Validation Flowchart

```mermaid
flowchart TD
    Start["1. Parse capabilities from event"] --> ForEach["2. For each capability"]
    ForEach --> A{"a. Is delegator the<br/>space controller?"}
    A -->|YES| Accept1["Accept<br/>(root authority)"]
    A -->|NO| B{"b. Parent delegations<br/>in proof chain?"}
    B -->|NO| Reject1["Reject<br/>(no justification)"]
    B -->|YES| C{"c. Parents provide broader<br/>or equal capabilities?"}
    C -->|NO| Reject2["Reject<br/>(capability escalation)"]
    C -->|YES| D{"d. Time bounds satisfied?<br/>(nbf ≤ now ≤ exp)"}
    D -->|NO| Reject3["Reject<br/>(outside valid time)"]
    D -->|YES| Accept2["Accept"]

    Accept1 --> Step3["3. Verify cryptographic signatures"]
    Accept2 --> Step3
    Step3 --> Step4["4. Check for revocations in proof chain"]
```

## D.2 Epoch DAG Structure

```mermaid
flowchart TD
    E0["Epoch 0<br/>seq: 0<br/><i>Genesis epoch (host delegation)</i>"]
    E1["Epoch 1<br/>seq: 1"]
    E2a["Epoch 2a<br/>seq: 2"]
    E2b["Epoch 2b<br/>seq: 2"]
    E3["Epoch 3<br/>seq: 3<br/><i>Merge epoch (convergence)</i>"]

    E0 --> E1
    E1 --> E2a
    E1 --> E2b
    E2a --> E3
    E2b --> E3

    subgraph fork["Concurrent epochs (fork)"]
        E2a
        E2b
    end
```

## D.3 Last-Write-Wins Conflict Resolution

```mermaid
flowchart TD
    Start["Conflict Resolution"] --> Step1["1. Compare epoch sequence numbers"]
    Step1 --> Check1{"Equal?"}
    Check1 -->|NO| Win1["Higher sequence wins"]
    Check1 -->|YES| Step2["2. Compare epoch CIDs"]
    Step2 --> Check2{"Equal?"}
    Check2 -->|NO| Win2["Higher CID<br/>(lexicographically) wins"]
    Check2 -->|YES| Step3["3. Compare event position within epoch"]
    Step3 --> Win3["Later position wins"]
```

## D.4 KV Store Write Process

```mermaid
sequenceDiagram
    participant Client
    participant Node as TinyCloud Node
    participant Storage

    Client->>Node: PUT /kv/path<br/>+ Authorization Header<br/>+ Body (value)

    Note over Node: 1. Stage value<br/>2. Compute hash

    Note over Node: 3. Parse token<br/>4. Validate chain

    Node->>Storage: Store value

    Note over Node: 5. Create KvWrite<br/>6. Create Epoch

    Node-->>Storage: Broadcast epoch to peers

    Node->>Client: 200 OK<br/>+ Version metadata
```

## D.5 Peer-to-Peer Replication Protocol

```mermaid
sequenceDiagram
    participant A as Host A
    participant B as Host B

    Note over A: Event received<br/>(delegation/invocation)

    Note over A: Validate & Create Epoch

    A->>B: Broadcast Epoch

    Note over B: Check parents known?

    alt Parents KNOWN
        Note over B: Validate & Apply
    else Parents UNKNOWN
        B->>A: Request missing epochs
        A->>B: Send epochs
        Note over B: Validate all bottom-up
    end
```

## D.6 Complete DAG Structure Example

```mermaid
flowchart TD
    subgraph EpochGenesis["Epoch (genesis)"]
        EG_content["parents: []<br/>events: [Link]"]
    end

    subgraph DelegationHost["Delegation (host)"]
        DH_content["delegator: did:key:z6MkRoot... (space controller)<br/>delegate: did:key:z6MkNode... (TinyCloud node)<br/>capabilities: tinycloud:.../hosts/* → tinycloud.space/host<br/>parents: [] (root authority)<br/>expiry: 1735689600"]
    end

    subgraph Epoch1["Epoch 1"]
        E1_content["parents: [Link → Epoch(genesis)]<br/>events: [Link → Delegation, [Link → Invocation, Link → KvWrite]]"]
    end

    subgraph DelegationSession["Delegation (session key)"]
        DS_content["delegator: Root<br/>delegate: Session<br/>capabilities: tinycloud.kv/*<br/>parents: []"]
    end

    subgraph InvocationKV["Invocation (kv write)"]
        IK_content["invoker: Session<br/>parents: [Link]<br/>capabilities: tinycloud.kv/put"]
    end

    subgraph KvWriteNode["KvWrite"]
        KW_content["key: data.json<br/>value: Link<br/>metadata: {...}<br/>version: {seq: 1, epoch: Hash, epochSeq: 1}"]
    end

    Content["Content<br/>(raw bytes)"]

    EpochGenesis --> DelegationHost
    DelegationHost --> Epoch1
    Epoch1 --> DelegationSession
    Epoch1 --> InvocationKV
    InvocationKV -.->|proof chain| DelegationSession
    InvocationKV --> KvWriteNode
    KvWriteNode --> Content
```
