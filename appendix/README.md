# TinyCloud Protocol: Technical Appendix

This document contains detailed technical specifications and background material for the TinyCloud Protocol. For the conceptual overview, see [whitepaper.md](../whitepaper.md).


---

## Table of Contents

### [Appendix A: Background Technologies](./appendix-a-background-technologies.md)
Provides foundational context on the key technologies TinyCloud builds upon, including Decentralized Identifiers (DIDs), User-Controlled Authorization Networks (UCANs), Sign-In with Ethereum (SIWE) with ReCaps, Chain Agnostic Capability Objects (CACAOs), the `did:pkh` method, and content addressing with IPLD.

### [Appendix B: TinyCloud URI ABNF Grammar](./appendix-b-uri-abnf-grammar.md)
Formally defines the syntax of string primitive types used in TinyCloud DAG nodes using ABNF grammar (RFC 5234), including DIDs, ResourceURIs, Abilities, and Paths.

### [Appendix C: IPLD Schema Definitions](./appendix-c-ipld-schema-definitions.md)
Specifies the IPLD schema definitions for TinyCloud's authorization state DAG, including core types, capability structures, event types (delegations, invocations, revocations), epoch structures, key-value operations, and SQL operations.

### [Appendix D: Detailed Diagrams](./appendix-d-detailed-diagrams.md)
Contains Mermaid diagrams illustrating key protocol flows and structures, including capability validation, epoch DAG structure, conflict resolution, KV store writes, peer-to-peer replication, and complete DAG structure examples.

### [Appendix E: SIWE ReCaps Example](./appendix-e-siwe-recaps-example.md)
Provides a concrete example of a Sign-In with Ethereum message with ReCaps capability delegation, along with TinyCloud URI examples for various resource types.

### [Appendix F: Glossary of Terms](./appendix-f-glossary.md)
Defines key terminology used throughout the TinyCloud Protocol documentation, including concepts like Autonomic Space, Capability, CACAO, Epoch, and UCAN.

### [Appendix G: Package Architecture](./appendix-g-package-architecture.md)
Documents the crate and package structure across TinyCloud's two repositories (tinycloud-node and web-sdk), including dependency relationships and deployment targets for backend servers and browser applications.

### [Appendix H: Delegation Protocol Specification](./appendix-h.md)
Specifies TinyCloud's delegation protocol including UCAN and CACAO formats, delegation chain validation rules, database schema, proof modes, and revocation semantics. Provides detailed examples of simple, chained, and attenuated delegations.

### [Appendix I: SDK Interface Specification](./appendix-i.md)
Defines the platform-agnostic SDK interface specification using pseudocode, covering the IUserAuthorization interface, Capability API, Delegation API, Space objects, Session management, Permission prompts, and Invocation chain building with concrete examples.

---

*This technical appendix accompanies the TinyCloud Protocol Whitepaper. For the conceptual overview, see [whitepaper.md](../whitepaper.md).*
