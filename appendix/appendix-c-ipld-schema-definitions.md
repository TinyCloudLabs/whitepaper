# Appendix C: IPLD Schema Definitions

TinyCloud's authorization state is encoded as an IPLD (InterPlanetary Linked Data) directed acyclic graph. This section formally defines the data structures using IPLD Schema notation.

## C.1 Core Type Definitions

```ipldsch
## TinyCloud Protocol IPLD Schema
## Version: 1.0

## ============================================================
## Primitive Types
## ============================================================

## A Content Identifier (CID) linking to another node in the DAG
type Link &Any

## Decentralized Identifier string (e.g., "did:key:z6Mk...")
type DID string

## TinyCloud Resource URI (e.g., "tinycloud:key:z6Mk.../kv/path")
type ResourceURI string

## Ability string following namespace/action format (e.g., "tinycloud.kv/put")
type Ability string

## Path within the key-value store
type Path string

## Unix timestamp in seconds
type Timestamp int

## Raw bytes (used for serialized tokens and content)
type Bytes bytes

## SHA-256 hash represented as bytes
type Hash bytes
```

## C.2 Capability Structure

```ipldsch
## ============================================================
## Capability Types
## ============================================================

## A capability associates a resource with an ability and optional caveats
type Capability struct {
    resource ResourceURI
    ability Ability
    caveats optional Caveats
} representation map

## Caveats constrain how a capability may be used
## The structure is ability-specific and extensible
type Caveats { String : Any }

## A set of capabilities grouped by resource
## Maps resource URIs to their associated abilities
type Capabilities { ResourceURI : AbilitySet }

## Set of abilities for a resource, each with optional caveats
type AbilitySet { Ability : [Caveats] }
```

## C.3 Event Types

```ipldsch
## ============================================================
## Authorization Events
## ============================================================

## Union of all event types that can appear in the DAG
type Event union {
    | Delegation "delegation"
    | Invocation "invocation"
    | Revocation "revocation"
} representation keyed

## ------------------------------------------------------------
## Delegation: Grants capabilities from one principal to another
## ------------------------------------------------------------

type Delegation struct {
    ## The principal granting the delegation (issuer)
    delegator DID

    ## The principal receiving the delegation (audience)
    delegate DID

    ## Capabilities being delegated
    capabilities Capabilities

    ## Parent delegations proving delegator's authority
    ## Empty if delegator is the namespace controller (root authority)
    parents [Link]

    ## Expiration timestamp (capability invalid after this time)
    expiry optional Timestamp

    ## Not-before timestamp (capability invalid before this time)
    notBefore optional Timestamp

    ## Issuance timestamp
    issuedAt optional Timestamp

    ## Arbitrary facts/metadata
    facts optional { String : Any }

    ## Original serialized token (UCAN JWT or CACAO)
    serialization Bytes
} representation map

## ------------------------------------------------------------
## Invocation: Exercises a capability to perform an action
## ------------------------------------------------------------

type Invocation struct {
    ## The principal performing the invocation
    invoker DID

    ## Capabilities being exercised
    capabilities Capabilities

    ## Delegation chain proving invoker's authority
    parents [Link]

    ## Timestamp when invocation was issued
    issuedAt Timestamp

    ## Arbitrary facts/metadata (e.g., request parameters)
    facts optional { String : Any }

    ## Original serialized token
    serialization Bytes
} representation map

## ------------------------------------------------------------
## Revocation: Invalidates a delegation
## ------------------------------------------------------------

type Revocation struct {
    ## The principal performing the revocation
    revoker DID

    ## Link to the delegation being revoked
    revoked Link

    ## Delegation chain proving revoker's authority to revoke
    ## Revoker must be in the original delegation's proof chain
    parents [Link]

    ## Original serialized revocation message
    serialization Bytes
} representation map
```

## C.4 Epoch Structure

```ipldsch
## ============================================================
## Epoch: Container for ordering events
## ============================================================

## An epoch is a signed container that orders events within the DAG
type Epoch struct {
    ## Links to parent epochs (enables DAG structure)
    ## Empty for the genesis epoch
    parents [Link]

    ## Events contained in this epoch
    ## Each entry is either a single event CID or multiple CIDs
    ## (invocations with side effects produce multiple linked nodes)
    events [EventEntry]
} representation map

## An event entry can be a single CID or multiple related CIDs
type EventEntry union {
    | Link "single"
    | [Link] "multiple"
} representation kinded

## Epoch metadata stored in the database
type EpochRecord struct {
    ## Hash-based identifier (derived from epoch contents)
    id Hash

    ## Monotonically increasing sequence number
    seq Int

    ## TinyCloud space this epoch belongs to
    space ResourceURI
} representation map
```

## C.5 Key-Value Operations

```ipldsch
## ============================================================
## Key-Value Store Operations
## ============================================================

## Metadata associated with a stored value
type Metadata { String : Any }

## A write operation to the key-value store
type KvWrite struct {
    ## TinyCloud space
    space ResourceURI

    ## Path/key being written
    key Path

    ## Link to the stored content
    value Link

    ## User-defined metadata
    metadata Metadata

    ## Version information for ordering
    version Version
} representation map

## A delete operation in the key-value store
type KvDelete struct {
    ## TinyCloud space
    space ResourceURI

    ## Path/key being deleted
    key Path

    ## Optional version specifier (deletes specific version)
    ## If absent, deletes the latest version
    version optional Version
} representation map

## Version information for ordering writes
type Version struct {
    ## Global sequence number
    seq Int

    ## Epoch hash this write belongs to
    epoch Hash

    ## Position within the epoch
    epochSeq Int
} representation map
```

## C.6 SQL Operations

```ipldsch
## ============================================================
## SQL Database Operations
## ============================================================

## A SQL query invocation
type SqlQuery struct {
    ## TinyCloud space
    space ResourceURI

    ## Database path (e.g., "myapp.db")
    database Path

    ## SQL statement to execute
    statement String

    ## Query parameters (for prepared statements)
    parameters [Any]

    ## Version of the database being queried
    version Version
} representation map

## Result of a SQL query
type SqlResult struct {
    ## TinyCloud space
    space ResourceURI

    ## Database path
    database Path

    ## Column names in result set
    columns [String]

    ## Row data (array of arrays)
    rows [[Any]]

    ## Number of rows affected (for INSERT/UPDATE/DELETE)
    rowsAffected optional Int
} representation map

## SQL capability caveats for table-level access
type SqlTableCaveat struct {
    ## Tables this capability applies to
    tables [String]

    ## Columns accessible (if omitted, all columns)
    columns optional [String]
} representation map

## SQL capability caveats for query-level access
type SqlQueryCaveat struct {
    ## Allowed prepared statements (exact match)
    statements [String]
} representation map
```

## C.7 Serialization Formats

TinyCloud nodes serialize DAG nodes using DAG-CBOR (Concise Binary Object Representation), which provides:

- **Deterministic encoding**: Same data always produces same bytes (required for content addressing)
- **Compact representation**: Efficient storage and transmission
- **IPLD compatibility**: Native support for CID links

| Node Type | IPLD Multicodec | CID Codec |
|-----------|-----------------|-----------|
| Epoch | dag-cbor | 0x71 |
| Delegation | raw | 0x55 |
| Invocation | raw | 0x55 |
| Revocation | raw | 0x55 |
| KvWrite | dag-cbor | 0x71 |
| SqlQuery | dag-cbor | 0x71 |
| SqlResult | dag-cbor | 0x71 |
| Content | raw | 0x55 |

Raw events (delegations, invocations, revocations) are stored as their original serialized bytes (JWT or CACAO format) to preserve signature validity.

## References

- IPLD Team, "IPLD Schemas," IPLD Specifications. [Online]. Available: https://ipld.io/specs/schemas/
- IPLD Team, "DAG-CBOR Specification," IPLD Specifications. [Online]. Available: https://ipld.io/specs/codecs/dag-cbor/spec/
