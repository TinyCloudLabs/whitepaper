# Appendix J: Services Specification

## Overview
TinyCloud organizes functionality into services, each providing a specific capability domain. Services are identified by URI prefixes and expose actions that can be delegated and invoked.

## Service Model

### Service URI Format
```
tinycloud.{service}/{action}
```

Examples:
- `tinycloud.kv/get` - Read from key-value store
- `tinycloud.sql/select` - Query SQL database
- `tinycloud.compute/execute` - Run compute function

### Capability Structure
```
{
  resource: "tinycloud:{did}:{space}/kv/photos/*",
  ability: "tinycloud.kv/get",
  caveats: { ... }
}
```

### Service Registration
Nodes declare supported services in their configuration:
```toml
[services]
kv = true
sql = false
compute = false
```

## Core Services

### KV Service (tinycloud.kv/*)

Key-value blob storage with versioning and metadata.

#### Actions
| Action | URI | Description |
|--------|-----|-------------|
| Get | `tinycloud.kv/get` | Read value and metadata |
| Put | `tinycloud.kv/put` | Write value with metadata |
| Delete | `tinycloud.kv/del` | Delete key |
| List | `tinycloud.kv/list` | List keys with prefix |
| Metadata | `tinycloud.kv/metadata` | Get metadata only |
| All | `tinycloud.kv/*` | All KV actions |

#### Caveats
```pseudocode
KvCaveats {
  path?: String       // Restrict to path prefix (e.g., "/photos/*")
  maxSize?: Number    // Maximum value size in bytes
  contentType?: String // Restrict content types
}
```

#### Implementation Status: COMPLETE

### SQL Service (tinycloud.sql/*)

Relational database access using SQLite.

#### Actions
| Action | URI | Description |
|--------|-----|-------------|
| Read | `tinycloud.sql/read` | Full read access |
| Write | `tinycloud.sql/write` | Full write access |
| Admin | `tinycloud.sql/admin` | Schema operations |
| Select | `tinycloud.sql/select` | Query with restrictions |
| Insert | `tinycloud.sql/insert` | Insert with table restrictions |
| Update | `tinycloud.sql/update` | Update with column restrictions |
| Delete | `tinycloud.sql/delete` | Delete with table restrictions |
| Execute | `tinycloud.sql/execute` | Run prepared statements |
| All | `tinycloud.sql/*` | All SQL actions |

#### Caveats
```pseudocode
SqlCaveats {
  tables?: [String]      // Allowed tables
  columns?: [String]     // Allowed columns (for select/update)
  statements?: [String]  // Allowed prepared statement names
  readOnly?: Boolean     // Force read-only mode
}
```

#### Implementation Status: SPEC COMPLETE, ENDPOINT NOT IMPLEMENTED

### Compute Service (tinycloud.compute/*)

Execute functions on namespace data with optional verifiable execution.

#### Actions
| Action | URI | Description |
|--------|-----|-------------|
| Execute | `tinycloud.compute/execute` | Run deployed function |
| Deploy | `tinycloud.compute/deploy` | Upload new function |
| List | `tinycloud.compute/list` | List deployed functions |
| All | `tinycloud.compute/*` | All compute actions |

#### Caveats
```pseudocode
ComputeCaveats {
  functions?: [String]   // Allowed function names
  maxDuration?: Number   // Maximum execution time (ms)
  maxMemory?: Number     // Maximum memory usage (bytes)
  inputs?: Schema        // Input validation schema
}
```

#### Function Types
- **WASM**: WebAssembly binaries
- **ZK VM**: Zero-knowledge provable execution (future)

#### Implementation Status: NOT IMPLEMENTED

### Space Service (tinycloud.space/*)

Space management and host authorization.

#### Actions
| Action | URI | Description |
|--------|-----|-------------|
| Host | `tinycloud.space/host` | Authorize node as host |
| Info | `tinycloud.space/info` | Get space metadata |

#### Implementation Status: PARTIAL (host delegation works)

### Capabilities Service (tinycloud.capabilities/*)

Query delegation authority.

#### Actions
| Action | URI | Description |
|--------|-----|-------------|
| Read | `tinycloud.capabilities/read` | List valid delegations |

#### Implementation Status: PARTIAL ("all" resource only)

## Service Discovery

### Phase 1: Not Supported Response (Current)

When a node receives a request for an unsupported service:

```http
POST /invoke
Authorization: <invocation with tinycloud.sql/read>

HTTP/1.1 501 Not Implemented
Content-Type: application/json

{
  "error": "service_not_supported",
  "service": "sql",
  "message": "This node does not support the SQL service",
  "supportedServices": ["kv", "namespace", "capabilities"],
  "hint": {
    "registry": "https://registry.tinycloud.xyz",
    "action": "Query registry for nodes supporting 'sql' service"
  }
}
```

### Phase 2: Registry-Based Discovery (Future)

#### Registry Format
```json
{
  "nodes": [
    {
      "id": "node-abc123",
      "endpoint": "https://node1.tinycloud.xyz",
      "services": ["kv", "sql"],
      "namespaces": ["*"],
      "publicKey": "did:key:z6Mk..."
    }
  ]
}
```

#### Discovery Query
```http
GET https://registry.tinycloud.xyz/discover?service=sql&namespace=ns1

{
  "nodes": [
    {
      "endpoint": "https://sql-node.tinycloud.xyz",
      "services": ["sql"],
      "latency": 45
    }
  ]
}
```

### Phase 3: Auto-Forwarding (Future)

Node configuration:
```toml
[services]
forward_unsupported = true
registry = "https://registry.tinycloud.xyz"
cache_ttl = 300  # seconds
```

Behavior: Instead of returning 501, node queries registry and forwards request transparently.

## Error Responses

### Standard Error Format
```json
{
  "error": "<error_code>",
  "message": "<human_readable>",
  "details": { ... }
}
```

### Error Codes
| Code | HTTP Status | Description |
|------|-------------|-------------|
| `service_not_supported` | 501 | Node doesn't support requested service |
| `capability_denied` | 403 | Invocation lacks required capability |
| `invalid_capability` | 400 | Malformed capability or caveat |
| `invalid_proof` | 400 | Delegation chain validation failed |
| `expired_delegation` | 403 | Delegation has expired |
| `revoked_delegation` | 403 | Delegation was revoked |
| `resource_not_found` | 404 | Requested resource doesn't exist |
| `quota_exceeded` | 429 | Storage or rate limit exceeded |

## Extensibility

### Custom Services
Third-party services can be added following the pattern:
```
tinycloud.{vendor}.{service}/{action}
```

Example: `tinycloud.acme.ml/inference`

### Service Versioning
Future versions may include version in URI:
```
tinycloud.kv.v2/get
```

Current services are implicitly v1.

Reference the main whitepaper and Appendix C (IPLD schemas) for data structure details.
