# Signatures Are All You Need: Cryptographic Access Control for AI and Applications

**Version 1.0**

**Authors:** Sam Gbafa, Charles Cunningham

---

## Table of Contents

1. [Abstract](#abstract)
2. [Introduction](#1-introduction)
3. [The Autonomic Space](#2-the-autonomic-space)
4. [Authorization Model](#3-authorization-model)
5. [Services](#4-services)
6. [Consistency & Replication](#5-consistency--replication)
7. [Future Directions](#6-future-directions)
8. [Security Considerations](#7-security-considerations)
9. [Conclusion](#8-conclusion)

For technical specifications, see the [Technical Appendix](./appendix/README.md).

---

## Abstract

Bitcoin let people hold and transfer value without banks. TinyCloud lets people hold and share data without platforms. Both rely on the same primitive: cryptographic signatures. Show up anywhere with your key. One signature proves ownership and unlocks access.

TinyCloud is a protocol for creating *spaces*—user-controlled data containers where individuals retain complete sovereignty over their information. Users delegate capabilities (read, write, compute, decrypt) to applications, devices, and AI agents using cryptographically signed tokens. Each delegation is self-verifying: the signature chain proves authorization without consulting external registries.

In an era of synthetic content and cloneable voices, cryptographic verifiability matters end-to-end. When AI systems operate on your data, you need proof of who authorized what. TinyCloud provides that signal—every access request carries a verifiable chain of signatures back to the data owner.

These authorization events form a hash-linked graph that replicates across trusted nodes, providing eventual consistency without centralized coordination. The result: portable, self-sovereign data that you control through explicit delegation rather than platform terms of service.

---

## 1. Introduction

As software development costs decrease, competitive advantages shift toward data ownership. AI tools now require structured, accessible data to function—you cannot ask an AI "what did I say yesterday?" without transcripts. This creates pressure to make personal and organizational data *legible*: structured, searchable, and machine-readable.

But information asymmetry is what preserves value. Coca-Cola's formula matters because the drink is good; a private key matters because the bitcoin it secures is worth something. Asymmetry protects what's already valuable. Traditional cloud architectures force a choice: make your data legible for AI and applications, or keep it private. Structuring data for utility typically means exposing it to platform operators, and the value leaks to whoever controls the infrastructure.

TinyCloud resolves this tension.

### The Legibility-Asymmetry Challenge

The core problem is that legibility and asymmetry appear to be in opposition:

- **Value exists first**: Data, knowledge, and assets have intrinsic worth—proprietary insights, personal records, creative work
- **Asymmetry preserves value**: Information asymmetry protects what's already valuable by controlling who can access it
- **Legibility creates new utility**: Making data structured enables AI and applications to operate on it, unlocking capabilities that weren't previously possible
- **The traditional trade-off**: Conventional systems require choosing between utility (legibility) and protection (asymmetry). If data is structured for AI, it becomes exposed

### How TinyCloud Resolves This

TinyCloud enables users to have both:

- **Full legibility**: Structure and capture everything—conversations, documents, metrics—for your own AI and application use
- **Preserved asymmetry**: Capability-based delegation means you control exactly who can access what, and under what conditions
- **Individual value capture**: The utility created by adding legibility while preserving asymmetry flows to the user, not the platform

Unlike traditional cloud architectures, legibility benefits you without benefiting adversaries or intermediaries.

TinyCloud provides:

- **Sovereignty**: Users maintain complete control and explicitly permission all access
- **Privacy**: Fine-grained permissions minimize data exposure
- **Interoperability**: Standard formats (JSON, SQLite, etc.) ensure portability

### Design Principles

- **All authority flows from the space controller**: The DID controls everything within the space
- **Explicit trust, not trustlessness**: Users authorize only computers they trust
- **Minimal trust requirements**: Delegated capabilities follow the principle of least authority
- **Eventual consistency**: Availability over strong consistency, with deterministic conflict resolution

### Why Now: Verifiability in the AI Era

As AI systems become more capable, two realities converge:

**AI needs access to your data.** Personal assistants, agents, and automated workflows require permission to read, write, and act on your behalf. Without a permission system, you either grant blanket access or get no utility.

**Synthetic content makes authenticity hard.** When voices can be cloned and text generated at scale, how do you know a request is genuine? The answer is signatures—cryptographic proof that a specific key authorized a specific action.

TinyCloud addresses both: it provides the permission infrastructure AI systems need to operate, while ensuring every access request carries verifiable proof of authorization. In a world of synthetic noise, cryptographic signatures are the signal.

---

## 2. The Autonomic Space

A TinyCloud space is *autonomic*—it describes how it is controlled within its own identifier. Any resource identified by a TinyCloud URI can be verified without consulting external authorities.

### URI Structure

```
tinycloud:pkh:eip155:1:0x6a12...C04B:default/kv/photos/vacation.jpg
└────────┘└─────────────────────────┘└─────┘└──┘└─────────────────┘
  scheme          DID suffix          space  service    path
```

| Component | Description |
|-----------|-------------|
| `tinycloud:` | Protocol identifier |
| DID suffix | The controlling identity (derived from a DID) |
| Space | User-defined subdivision (e.g., "default", "work") |
| Service | The service type (e.g., "kv", "compute", "capabilities") |
| Path | Resource path within the service |

### Relationship to DIDs

The TinyCloud URI maintains a bijective relationship with Decentralized Identifiers (DIDs). To construct a TinyCloud URI: take a DID, replace `did:` with `tinycloud:`, and append the space and path. This ensures all resources are automatically protected by the DID's authorization chain.

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

The space controller (root DID) has absolute authority. They delegate capabilities to session keys, applications, or other users. Each delegation can be *attenuated*—granting narrower permissions than held. Delegations can include time bounds (expiry, not-before) and path restrictions.

### Session Keys

A common pattern is delegating to an ephemeral "session key" generated for a specific session or device. The root wallet signs once to authorize the session key, which then handles all subsequent operations without repeated wallet interactions.

### Policy Engine

Capabilities can be delegated to a policy engine that grants conditional access. Instead of granting direct access, you delegate to a policy that evaluates conditions (location, time, attestations) before authorizing the action. This enables dynamic access control without changing the underlying delegation structure.

---

## 4. Services

TinyCloud provides services through the URI path structure. Each service defines its own capabilities and semantics within a space.

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

Execute functions on data within the space:

| Ability | Description |
|---------|-------------|
| `tinycloud.compute/execute` | Run a function with specified inputs |
| `tinycloud.compute/deploy` | Upload a new function |

Functions are WebAssembly binaries (or ZK VM programs for verifiable execution). You delegate permission to execute a specific function on specific data, and the result can be stored in TinyCloud or returned directly.

### Encryption (`encryption`)

Threshold decryption and proxy re-encryption for data sharing:

| Ability | Description |
|---------|-------------|
| `tinycloud.encryption/encrypt` | Encrypt data to the space |
| `tinycloud.encryption/decrypt` | Request decryption (via threshold network) |
| `tinycloud.encryption/reencrypt` | Proxy re-encrypt to another recipient |

Data is encrypted client-side before storage. TinyCloud nodes participate in threshold decryption—no single node can decrypt unilaterally. Proxy re-encryption enables sharing without exposing plaintext to intermediaries.

### SQL Database (`sql`)

Relational database storage using SQLite:

| Ability | Description |
|---------|-------------|
| `tinycloud.sql/read` | Full read access (any SELECT) |
| `tinycloud.sql/write` | Full write access (INSERT, UPDATE, DELETE) |
| `tinycloud.sql/admin` | Schema changes (CREATE, ALTER, DROP) |
| `tinycloud.sql/select` | SELECT with table/column restrictions |
| `tinycloud.sql/execute` | Execute specific prepared statements |

SQLite databases are stored as files within the space at `sql/<database-name>`. Permissions can be scoped hierarchically:

- **Database level**: Full read/write/admin access to the entire database
- **Table level**: Access to specific tables and columns via caveats
- **Query level**: Execute only specific prepared statements

```
tinycloud:pkh:eip155:1:0x6a12...C04B:default/sql/myapp.db
└────────────────────────────────────────────┘└──┘└────────┘
                  space                       service  database
```

This enables applications to use SQL for relational data while maintaining TinyCloud's capability-based authorization. Applications can choose KV for blob storage or SQL for structured queries—both with the same authorization model.

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
- **Manifest service**: A registry mapping spaces to hosts

When events occur, hosts broadcast epochs to peers. Peers validate the epoch chain and apply events. Missing history is requested and validated bottom-up from genesis.

---

## 6. Future Directions

### Verifiable Compute with ZK VMs

Zero-knowledge virtual machines (RISC Zero, SP1) enable cryptographic proofs that computations executed correctly. For TinyCloud, this means:

1. **Verifiable Functions**: Execute WASM or native code in a ZK VM, producing a proof alongside the result
2. **Trustless Verification**: Anyone can verify `verify(proof, function_cid, inputs, outputs) → bool` without re-executing
3. **Computation Delegation**: Delegate compute capability to untrusted nodes that must provide proofs

Current ZK VMs achieve near-real-time proving for complex workloads—Ethereum blocks can be proven in under a minute with systems like R0VM 2.0 and SP1 Hypercube. This makes verifiable compute practical for TinyCloud applications requiring auditability or cross-party trust.

### Verifiable Authorization

ZK proofs can extend beyond compute to authorization itself:

- **Delegation Validation Proof**: Prove that a delegation was correctly validated against space policy
- **Epoch Transition Proof**: Prove that epoch N+1 correctly follows from epoch N
- **State Root Proof**: Prove current state root derives from genesis through valid transitions

This enables light clients to verify TinyCloud state with a single proof instead of replaying the full DAG.

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

The security of a TinyCloud space depends on protecting the controller's private key. Consider hardware security modules, threshold signatures for high-value spaces, and proper key rotation procedures.

### Trusted Execution Environments

The security model above ensures **integrity**—hosts cannot forge delegations or modify data undetectably. However, without additional measures, a host could observe plaintext data during processing.

For hosted TinyCloud nodes, **Trusted Execution Environments (TEEs)** provide confidentiality guarantees:

- Data remains encrypted in memory during processing
- Node operators cannot access plaintext, even with physical access
- Remote attestation allows users to verify the execution environment before delegating

This extends trust from the user's private key to the compute environment. Users self-hosting TinyCloud control their own hardware; users delegating to hosted nodes rely on TEE attestation to ensure their data remains confidential.

TinyCloud nodes are currently deployed using DStack on Phala Network, which provides TEE-based confidential computing. The protocol is designed to be TEE-agnostic, supporting any environment that provides attestation and memory encryption (e.g., Intel SGX, Intel TDX, AMD SEV).

Alternative approaches like threshold encryption and homomorphic encryption offer different tradeoffs. TEEs provide practical confidential compute for real workloads today, while these cryptographic approaches may become viable as performance improves.

---

## 8. Conclusion

TinyCloud provides self-sovereign data control through:

- **Autonomic spaces** that encode their own authorization semantics
- **Capability-based security** with delegable, attenuable permissions
- **Epoch-based consistency** through hash-linked DAGs
- **Extensible services** for storage, compute, and encryption

Unlike systems relying on economic incentives, TinyCloud operates on explicit trust—users authorize only hosts they trust. This provides strong security while enabling high performance and offline-first operation.

The protocol demonstrates that user sovereignty over data is achievable without sacrificing interoperability or requiring complex infrastructure.

---

## Technical Appendix

For detailed specifications including IPLD schemas, ABNF grammars, and protocol diagrams, see the [Technical Appendix](./appendix/README.md).

---

*This whitepaper is based on the TinyCloud Protocol implementation. For the reference implementation, see the TinyCloud Protocol repository.*
