# TinyCloud Protocol: A Self-Sovereign Autonomic Namespace for Decentralized Data Control

**Version 2.0**

**Authors:** Charles Cunningham, Sam Gbafa

---

## Table of Contents

1. [Abstract](#abstract)
2. [Introduction](#1-introduction)
3. [The Autonomic Namespace](#2-the-autonomic-namespace)
4. [Authorization Model](#3-authorization-model)
5. [Services](#4-services)
6. [Consistency & Replication](#5-consistency--replication)
7. [Future Directions](#6-future-directions)
8. [Security Considerations](#7-security-considerations)
9. [Conclusion](#8-conclusion)

For technical specifications, see the [Technical Appendix](./appendix.md).

---

## Abstract

TinyCloud is a protocol for creating user-controlled data namespaces where individuals retain complete sovereignty over their information. A TinyCloud namespace is *autonomic*—its URI encodes how it is controlled, enabling any participant to verify authorization without external registries. Users delegate capabilities (read, write, compute, decrypt) to applications and devices using cryptographically signed tokens. These authorization events form a hash-linked graph that replicates across trusted nodes, providing eventual consistency without centralized coordination. The result is portable, self-sovereign data that users control through explicit delegation rather than platform terms of service.

---

## 1. Introduction

As software development costs decrease, competitive advantages shift toward data ownership. Traditional cloud architectures require users to cede control of their data to third parties, creating privacy concerns and vendor lock-in.

TinyCloud addresses this by providing:

- **Sovereignty**: Users maintain complete control and explicitly permission all access
- **Privacy**: Fine-grained permissions minimize data exposure
- **Interoperability**: Standard formats (JSON, SQLite, etc.) ensure portability

### Design Principles

- **All authority flows from the namespace controller**: The DID controls everything within the namespace
- **Explicit trust, not trustlessness**: Users authorize only computers they trust
- **Minimal trust requirements**: Delegated capabilities follow the principle of least authority
- **Eventual consistency**: Availability over strong consistency, with deterministic conflict resolution

---

## 2. The Autonomic Namespace

A TinyCloud namespace is *autonomic*—it describes how it is controlled within its own identifier. Any resource identified by a TinyCloud URI can be verified without consulting external authorities.

### URI Structure

```
tinycloud:pkh:eip155:1:0x6a12...C04B:default/kv/photos/vacation.jpg
└────────┘└───────────────────────────┘└─────┘└──┘└───────────────┘
  scheme          DID suffix         namespace service    path
```

| Component | Description |
|-----------|-------------|
| `tinycloud:` | Protocol identifier |
| DID suffix | The controlling identity (derived from a DID) |
| Namespace | User-defined subdivision (e.g., "default", "work") |
| Service | The service type (e.g., "kv", "compute", "capabilities") |
| Path | Resource path within the service |

### Relationship to DIDs

The TinyCloud URI maintains a bijective relationship with Decentralized Identifiers (DIDs). To construct a TinyCloud URI: take a DID, replace `did:` with `tinycloud:`, and append the namespace and path. This ensures all resources are automatically protected by the DID's authorization chain.

---

## 3. Authorization Model

TinyCloud uses capability-based access control through three types of cryptographically signed events:

### Event Types

| Event | Purpose |
|-------|---------|
| **Delegation** | Grants capabilities from one principal to another |
| **Invocation** | Exercises a capability to perform an action |
| **Revocation** | Invalidates a delegation and all derived capabilities |

### Authority Flow

```
┌─────────────────┐      delegate      ┌─────────────────┐      invoke      ┌─────────────────┐
│  Root DID       │ ─────────────────► │  Session Key    │ ───────────────► │  Resource       │
│  (Wallet)       │                    │  (Browser)      │                  │  (KV Store)     │
└─────────────────┘                    └─────────────────┘                  └─────────────────┘
```

The namespace controller (root DID) has absolute authority. They delegate capabilities to session keys, applications, or other users. Each delegation can be *attenuated*—granting narrower permissions than held. Delegations can include time bounds (expiry, not-before) and path restrictions.

### Session Keys

A common pattern is delegating to an ephemeral "session key" generated for a specific session or device. The root wallet signs once to authorize the session key, which then handles all subsequent operations without repeated wallet interactions.

### Policy Engine

Capabilities can be delegated to a policy engine that grants conditional access. Instead of granting direct access, you delegate to a policy that evaluates conditions (location, time, attestations) before authorizing the action. This enables dynamic access control without changing the underlying delegation structure.

---

## 4. Services

TinyCloud provides services through the URI path structure. Each service defines its own capabilities and semantics.

### Key-Value Store (`kv`)

S3-like blob storage with version history:

| Ability | Description |
|---------|-------------|
| `tinycloud.kv/get` | Read value at path |
| `tinycloud.kv/put` | Write value at path |
| `tinycloud.kv/del` | Delete value at path |
| `tinycloud.kv/list` | List paths under prefix |

Every write creates a new version. Previous versions remain accessible. Conflicts are resolved deterministically using Last-Write-Wins semantics based on epoch ordering.

### Compute (`compute`)

Execute functions on data within the namespace:

| Ability | Description |
|---------|-------------|
| `tinycloud.compute/execute` | Run a function with specified inputs |
| `tinycloud.compute/deploy` | Upload a new function |

Functions are WebAssembly binaries (or ZK VM programs for verifiable execution). You delegate permission to execute a specific function on specific data, and the result can be stored in TinyCloud or returned directly.

### Encryption (`encryption`)

Threshold decryption and proxy re-encryption for data sharing:

| Ability | Description |
|---------|-------------|
| `tinycloud.encryption/encrypt` | Encrypt data to the namespace |
| `tinycloud.encryption/decrypt` | Request decryption (via threshold network) |
| `tinycloud.encryption/reencrypt` | Proxy re-encrypt to another recipient |

Data is encrypted client-side before storage. TinyCloud nodes participate in threshold decryption—no single node can decrypt unilaterally. Proxy re-encryption enables sharing without exposing plaintext to intermediaries.

### Service Extensibility

New services follow the pattern `tinycloud.<service>/<action>`. This extensibility allows the protocol to grow while maintaining consistent authorization semantics.

---

## 5. Consistency & Replication

### Epoch-Based Ordering

TinyCloud maintains consistency through *epochs*—signed containers that order events in a hash-linked DAG:

```
┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐
│ Epoch 0  │ ──► │ Epoch 1  │ ──► │ Epoch 2a │ ──► │ Epoch 3  │
│ (genesis)│     │          │     │          │     │ (merge)  │
└──────────┘     └──────────┘     │          │     └──────────┘
                      │           └──────────┘          ▲
                      │                                 │
                      │           ┌──────────┐          │
                      └─────────► │ Epoch 2b │ ─────────┘
                                  │          │
                                  └──────────┘
```

Each epoch has a sequence number and parent references. When forks occur (concurrent writes), they merge deterministically. The Last-Write-Wins CRDT resolves conflicts: higher sequence wins; ties break by lexicographic CID comparison.

### Peer-to-Peer Replication

Hosts discover each other through:
- **Host delegations**: Including multi-addresses in the delegation
- **DID documents**: Service endpoints advertising node locations
- **Manifest service**: A registry mapping namespaces to hosts

When events occur, hosts broadcast epochs to peers. Peers validate the epoch chain and apply events. Missing history is requested and validated bottom-up from genesis.

---

## 6. Future Directions

### Verifiable Compute with ZK VMs

Zero-knowledge virtual machines (RISC-V based) can provide cryptographic proofs that computations executed correctly. This enables trustless verification: given a TinyCloud state hash and a computation result, anyone can verify correctness without re-executing.

### On-Chain State Anchoring

Posting state roots on-chain (as an L2/L3) enables light client verification. Clients download a single hash, request a ZK proof, and verify they have the complete, correct view—without trusting individual nodes or downloading full history.

### Light Client Verification

With recursive ZK proofs, clients can verify authorization chains without storing the full DAG. This enables browser-based TinyCloud instances that maintain security guarantees with minimal local storage.

---

## 7. Security Considerations

### Trust Model

TinyCloud operates on **explicit trust** rather than economic trustlessness:

| Aspect | TinyCloud | Blockchain Systems |
|--------|-----------|-------------------|
| **Trust Basis** | Explicit delegation | Economic incentives |
| **Node Authorization** | Must receive host delegation | Permissionless |
| **Security Guarantee** | As strong as private key protection | Consensus mechanism |

A malicious host could refuse to serve data or fail to propagate updates, but cannot forge delegations or modify data undetectably.

### Capability Security

Following the Principle of Least Authority (POLA):
- Grant only specific capabilities needed
- Use time bounds (expiry, not-before)
- Prefer session keys over sharing root keys
- Revoke promptly when access is no longer needed

### Key Management

The security of a TinyCloud namespace depends on protecting the controller's private key. Consider hardware security modules, threshold signatures for high-value namespaces, and proper key rotation procedures.

---

## 8. Conclusion

TinyCloud provides self-sovereign data control through:

- **Autonomic namespaces** that encode their own authorization semantics
- **Capability-based security** with delegable, attenuable permissions
- **Epoch-based consistency** through hash-linked DAGs
- **Extensible services** for storage, compute, and encryption

Unlike systems relying on economic incentives, TinyCloud operates on explicit trust—users authorize only hosts they trust. This provides strong security while enabling high performance and offline-first operation.

The protocol demonstrates that user sovereignty over data is achievable without sacrificing interoperability or requiring complex infrastructure.

---

## Technical Appendix

For detailed specifications including IPLD schemas, ABNF grammars, and protocol diagrams, see the [Technical Appendix](./appendix.md).

---

*This whitepaper is based on the TinyCloud Protocol implementation. For the reference implementation, see the TinyCloud Protocol repository.*
