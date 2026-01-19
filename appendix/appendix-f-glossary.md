# Appendix F: Glossary of Terms

| Term | Definition |
|------|------------|
| **Autonomic Space** | A space that is self-certifying and self-administrating—it contains within its identifier the cryptographic material needed to verify control authority |
| **Capability** | A token representing the right to perform a specific action on a specific resource; in TinyCloud: `resource × ability × caveats` |
| **CACAO** | Chain Agnostic CApability Object—an IPLD container for signed capability delegations (CAIP-74) |
| **Caveat** | A constraint that limits how a capability may be used (e.g., time bounds, path restrictions) |
| **CID** | Content Identifier—a self-describing hash used to reference content-addressed data |
| **DAG** | Directed Acyclic Graph—the data structure used for epochs and authorization state |
| **Delegation** | An authorization event that grants capabilities from one principal to another |
| **DID** | Decentralized Identifier—a globally unique identifier not requiring centralized registration |
| **Epoch** | A signed container of events that establishes ordering in the TinyCloud DAG |
| **Host** | A TinyCloud node authorized to receive events, create epochs, and replicate with peers |
| **Invocation** | An authorization event that exercises a capability to perform an action |
| **IPLD** | InterPlanetary Linked Data—a data model for content-addressed linked data structures |
| **LWW CRDT** | Last-Write-Wins Conflict-free Replicated Data Type—the consistency model for the KV store |
| **Space Controller** | The DID that has root authority over a TinyCloud space |
| **POLA** | Principle of Least Authority—granting only the minimum capabilities needed |
| **Revocation** | An authorization event that invalidates a delegation |
| **Session Key** | An ephemeral key pair delegated capabilities for a specific session or device |
| **SIWE** | Sign-In with Ethereum—a standard for authenticating with Ethereum accounts (EIP-4361) |
| **SQL Service** | TinyCloud service providing SQLite database storage with capability-based access control |
| **SQLite** | A lightweight, embedded relational database stored as a single file |
| **Table Caveat** | A capability restriction limiting SQL access to specific tables and columns |
| **Query Caveat** | A capability restriction limiting SQL access to specific prepared statements |
| **ReCaps** | SIWE capability extensions enabling UCAN-like semantics (EIP-5573) |
| **UCAN** | User-Controlled Authorization Network—the capability delegation scheme used by TinyCloud |
| **ZK VM** | Zero-Knowledge Virtual Machine—enables verifiable computation with cryptographic proofs |
