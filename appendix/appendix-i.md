# Appendix I: SDK Interface Specification

## Overview

The TinyCloud SDK provides a unified interface for interacting with TinyCloud spaces across platforms. This specification uses pseudocode to define platform-agnostic interfaces that both web and node implementations must follow.

## Design Philosophy

The SDK is designed around four core principles:

1. **Two-layer architecture**: A high-level convenience API handles common operations automatically, while a low-level UCAN escape hatch provides direct access to the underlying capability primitives.

2. **Platform isolation**: Platform-specific concerns (wallet popups, CLI prompts, key management) are isolated within the `IUserAuthorization` interface, enabling consistent business logic across environments.

3. **Capability-first delegation**: All operations are modeled as capability invocations. The SDK manages delegation chains, proof construction, and capability attenuation transparently.

4. **Automatic complexity management**: Session creation, key rotation, and proof chain construction happen automatically by default, with explicit overrides available for advanced use cases.

## IUserAuthorization Interface

The `IUserAuthorization` interface abstracts platform-specific user interaction patterns. Implementations must provide mechanisms for signing messages and approving capability requests.

### Core Methods

```pseudocode
interface IUserAuthorization {
  // Sign a message with the user's controlling key
  signMessage(message: String) -> Result<Signature, Error>

  // Request user approval for a capability grant
  approveCapability(capability: Capability) -> Result<Approval, Rejection>

  // Retrieve the user's DID
  getUserDID() -> DID

  // Check if user can grant a specific capability
  canGrant(capability: Capability) -> Boolean
}

type Approval = { success: true }
type Rejection = { success: false, reason: String }
```

### WebUserAuthorization

The web implementation leverages browser wallet extensions for cryptographic operations and user approval.

```pseudocode
class WebUserAuthorization implements IUserAuthorization {
  provider: WalletProvider  // MetaMask, WalletConnect, etc.

  signMessage(message):
    // Triggers wallet popup for SIWE message signing
    // Message includes ReCap extensions for capability encoding
    return provider.signMessage(message)

  approveCapability(capability):
    // Display modal dialog describing the requested capability
    // Returns user's decision
    return showCapabilityModal(capability)

  getUserDID():
    address = provider.getAddress()
    chainId = provider.getChainId()
    return DID.fromPKH("eip155", chainId, address)

  canGrant(capability):
    // User can grant if they own the space
    return capability.space.owner == getUserDID()
}
```

### NodeUserAuthorization

The node implementation supports multiple authorization strategies for different deployment scenarios.

```pseudocode
class NodeUserAuthorization implements IUserAuthorization {
  strategy: AuthStrategy
  config: AuthConfig

  constructor(config: {
    strategy: "auto-sign" | "auto-reject" | "callback" | "event-emitter"
    signer?: PrivateKey          // Required for auto-sign
    handler?: Function           // Required for callback
    emitter?: EventEmitter       // Required for event-emitter
  })
}
```

**Strategy Descriptions:**

- **auto-sign**: Automatically approves all capability requests and signs with the provided private key. Suitable for trusted backend services that operate with pre-authorized capabilities.

- **auto-reject**: Rejects all capability requests. Useful for read-only deployments or scenarios where capabilities are pre-delegated.

- **callback**: Invokes a user-provided function for each authorization request. The function receives the capability and returns an approval decision.

- **event-emitter**: Emits events for authorization requests and awaits responses. Designed for CLI integration where human input is expected.

```pseudocode
// Auto-sign strategy for backend services
nodeAuth = NodeUserAuthorization({
  strategy: "auto-sign",
  signer: loadPrivateKeyFromEnv()
})

// Callback strategy for custom logic
nodeAuth = NodeUserAuthorization({
  strategy: "callback",
  handler: function(capability) {
    if capability.isRead():
      return Approval()
    else:
      return Rejection("Write operations disabled")
  }
})

// Event-emitter strategy for CLI
nodeAuth = NodeUserAuthorization({
  strategy: "event-emitter",
  emitter: cliEventEmitter
})
// CLI listens for 'capability-request' events and emits 'capability-response'
```

## Capability API

Capabilities represent specific permissions within TinyCloud spaces. The SDK provides a fluent builder API for constructing capabilities, with direct access to underlying UCAN structures when needed.

### High-Level Capability Builder

The builder pattern allows readable capability construction with type safety.

```pseudocode
// Key-Value Store Capabilities
Capability.kv()
  .read()                     // Grants tinycloud.kv/get
  .write()                    // Grants tinycloud.kv/put
  .delete()                   // Grants tinycloud.kv/del
  .list()                     // Grants tinycloud.kv/list
  .all()                      // Grants tinycloud.kv/*
  .atPath(path: String)       // Restricts to path prefix
  .expiring(duration: String) // Time limit: "1h", "7d", "30m"
  .notBefore(time: DateTime)  // Earliest activation time

// SQL Capabilities
Capability.sql()
  .read()                                 // Grants tinycloud.sql/read
  .write()                                // Grants tinycloud.sql/write
  .select(tables: [String])               // Grants tinycloud.sql/select with table restriction
  .insert(tables: [String])               // Grants tinycloud.sql/insert with table restriction
  .update(tables: [String], columns?: [String])  // With optional column restriction
  .delete(tables: [String])               // Grants tinycloud.sql/delete with table restriction
  .execute(statements: [String])          // Grants specific statement execution

// Compute Capabilities
Capability.compute()
  .execute(functionName: String)  // Grants execution of specific function
  .deploy()                       // Grants deployment rights
  .all()                          // Grants tinycloud.compute/*
```

### Capability Composition

Multiple capability restrictions can be chained.

```pseudocode
// Read-only access to photos, expiring in 1 hour
photoReadCap = Capability.kv()
  .read()
  .list()
  .atPath("/photos")
  .expiring("1h")

// Write to specific tables with column restrictions
userUpdateCap = Capability.sql()
  .update(["users"], ["display_name", "avatar_url"])
  .expiring("24h")
```

### Low-Level UCAN Access

For advanced use cases, capabilities can be converted to and from raw UCAN structures.

```pseudocode
// Convert high-level capability to raw UCAN
ucan = capability.toUCAN()

// Inspect UCAN structure
ucan.issuer      // DID of the granting party
ucan.audience    // DID of the receiving party
ucan.expiration  // Unix timestamp
ucan.attenuation // Array of capability objects
ucan.proofs      // CIDs of parent delegations

// Create capability from existing UCAN
capability = Capability.fromUCAN(ucan)

// Access internal structure directly
capability.resource    // TinyCloud URI (e.g., tinycloud:pkh:eip155:1:0x.../photos)
capability.ability     // Action string (e.g., tinycloud.kv/get)
capability.caveats     // Restriction object
```

## Delegation API

Delegation enables capability transfer between parties. The SDK manages the full delegation lifecycle: creation, transmission, storage, and revocation.

### Granting Capabilities

```pseudocode
// Simple grant
result = tc.grant(capability).to(delegateDID)

// Grant with additional options
result = tc.grant(capability)
  .to(delegateDID)
  .notBefore(datetime)           // Delayed activation
  .expiring(datetime)            // Explicit expiration
  .withFacts({ purpose: "photo editing" })  // Metadata
  .send()

// Result types
type GrantResult =
  | { success: true, delegation: Delegation, cid: CID }
  | { success: false, reason: String }
```

The `send()` method transmits the delegation to the TinyCloud node, which stores it and makes it available in the recipient's inbox.

### Receiving Delegations (Inbox)

The inbox provides access to delegations received from other parties.

```pseudocode
// Synchronize inbox with node
tc.inbox.sync()

// List all received delegations
delegations = tc.inbox.list()

// Filter by service type
delegations = tc.inbox.list({ service: "kv" })
delegations = tc.inbox.list({ service: "sql" })

// Filter by granting party
delegations = tc.inbox.list({ from: aliceDID })

// Filter by action
delegations = tc.inbox.list({ action: "read" })
delegations = tc.inbox.list({ action: "write" })

// Retrieve specific delegation
delegation = tc.inbox.get(delegationId)

// Cache management
tc.inbox.isCached(delegationId) -> Boolean
tc.inbox.clearCache()
```

### Re-delegation

Received delegations can be further delegated to other parties, subject to the original capability's constraints.

```pseudocode
// Re-delegate with attenuation (narrower scope)
result = tc.redelegate(receivedDelegation)
  .to(anotherDID)
  .attenuate({ path: "/photos/public" })  // Restrict to subdirectory
  .expiring("1h")                          // Shorter than original
  .send()

// Attenuation rules:
// - Can only narrow, never expand, the original capability
// - Expiration cannot exceed the original delegation's expiration
// - Path restrictions must be subsets of the original path
```

## Space Objects

Space objects provide a scoped interface to a specific TinyCloud space. All operations through a space object are automatically scoped to that space's resources.

```pseudocode
// Get a space object
sp = tc.space(spaceId)

// Scoped storage operations
sp.storage.get(key: String) -> Result<Bytes, Error>
sp.storage.put(key: String, value: Bytes, options?: PutOptions) -> Result<CID, Error>
sp.storage.delete(key: String) -> Result<Unit, Error>
sp.storage.list(prefix?: String) -> Result<[String], Error>
sp.storage.deleteAll(prefix?: String) -> Result<Count, Error>

// Scoped SQL operations
sp.sql.query(statement: String, params?: [Value]) -> Result<Rows, Error>
sp.sql.execute(statement: String, params?: [Value]) -> Result<AffectedCount, Error>

// Scoped compute operations
sp.compute.invoke(functionName: String, input: Bytes) -> Result<Bytes, Error>
sp.compute.deploy(wasmBinary: Bytes) -> Result<FunctionId, Error>

// Scoped delegation (grants within this space)
sp.grant(capability).to(did)

// Space metadata
info = sp.info()
// Returns: { id: SpaceId, owner: DID, created: DateTime, hosts: [DID] }
```

### Default Space

For convenience, the SDK provides top-level methods that operate on a default space.

```pseudocode
// Configure default space
tc = TinyCloud({
  defaultSpaceId: "my-app-space"
})

// These are equivalent
tc.storage.get(key)
tc.space("my-app-space").storage.get(key)
```

## Session Management

Sessions enable delegated access without requiring repeated wallet signatures. The SDK manages session lifecycle automatically by default.

### Automatic Session Management

By default, the SDK handles session creation and rotation without explicit user action.

```pseudocode
// Sessions are created automatically on first operation
result = tc.storage.get("/photos/vacation.jpg")
// If no valid session exists:
//   1. SDK generates a new ed25519 keypair
//   2. Requests user approval for capability grant
//   3. Stores session locally
//   4. Proceeds with operation
```

### Explicit Session Control

For advanced use cases, sessions can be managed explicitly.

```pseudocode
// Create a session with specific capabilities and duration
session = tc.createSession({
  purpose: "photo-editor",
  capabilities: [
    Capability.kv().read().atPath("/photos"),
    Capability.kv().write().atPath("/photos/edits")
  ],
  expiry: "1h"
})

// Session properties
session.id           // Unique identifier
session.did          // Session key DID
session.capabilities // Granted capabilities
session.expiresAt    // Expiration timestamp
session.createdAt    // Creation timestamp
```

### Session Lifecycle Hooks

Applications can respond to session events.

```pseudocode
// Called when a new session is created
tc.onSessionCreated(function(session) {
  log("New session created:", session.id)
})

// Called before session expires (default: 5 minutes before)
tc.onSessionExpiring(function(session, remainingTime) {
  log("Session expiring in:", remainingTime)
  // SDK will attempt automatic rotation if autoRotate is enabled
})

// Called after session rotation completes
tc.onSessionRotated(function(oldSession, newSession) {
  log("Session rotated from", oldSession.id, "to", newSession.id)
})
```

## Permission Prompts

When an operation requires capabilities the user hasn't granted, the SDK can automatically prompt for permission using an iOS-style progressive disclosure pattern.

### Configuration

```pseudocode
tc = TinyCloud({
  permissions: {
    autoPrompt: true,        // Enable automatic prompts (default: true)
    promptUI: "modal",       // "modal" | "toast" | "custom"
    customPrompt: function   // Custom prompt implementation (for "custom" mode)
  }
})
```

### Prompt Behavior

When an operation fails due to missing capabilities:

1. SDK checks if the user can grant the required capability
2. If `autoPrompt` is enabled and user can grant, displays prompt
3. User approves or declines the request
4. If approved, SDK creates delegation and retries operation
5. If declined, SDK returns a result (not an exception)

```pseudocode
// Attempting to write without write capability
result = tc.storage.put("/photos/new.jpg", imageData)

// If user hasn't granted write access:
// - Modal appears: "Allow 'My App' to save files to your photos?"
// - User approves or declines
// - Result reflects the outcome
```

### Return Value on Decline

When the user declines a permission request, the SDK returns a structured result rather than throwing an exception.

```pseudocode
type PermissionResult =
  | { success: true, data: T }
  | {
      success: false,
      reason: "user_declined" | "not_authorized" | "no_capability",
      capability: Capability,  // The capability that was needed
      operation: String        // The operation that was attempted
    }

// Example handling
result = tc.storage.put("/photos/new.jpg", imageData)
if not result.success:
  if result.reason == "user_declined":
    showMessage("Permission required to save photos")
  else if result.reason == "not_authorized":
    showMessage("You don't have access to this space")
```

## Invocation Chain Building

Every operation in TinyCloud is an invocation that requires a proof chain demonstrating the invoker's authority. The SDK constructs these chains automatically.

### Automatic Chain Building

```pseudocode
// SDK automatically tracks delegations and builds proof chains
result = ns.storage.get(key)

// Internally, the SDK:
// 1. Identifies the required capability (tinycloud.kv/get for this key)
// 2. Searches inbox for covering delegation
// 3. Constructs minimal proof chain
// 4. Attaches chain to invocation
```

### Manual Override

For advanced scenarios, proof chains can be specified explicitly.

```pseudocode
// Specify exact proof chain
result = ns.storage.get(key, {
  proof: [delegation1, delegation2]
})
```

### Chain Building Logic

The SDK uses the following algorithm to construct proof chains:

```pseudocode
function buildProofChain(capability, invokerDID) -> [CID]:
  // Case 1: Invoker owns the space
  if capability.space.owner == invokerDID:
    return []  // Owner needs no proof

  // Case 2: Direct delegation exists
  direct = inbox.find(d =>
    d.delegatee == invokerDID &&
    d.capability.covers(capability)
  )
  if direct:
    return [direct.cid]

  // Case 3: Transitive delegation (chain required)
  chain = []
  current = capability
  while current != null:
    delegation = inbox.find(d =>
      d.capability.covers(current) &&
      canReach(d.delegator, capability.space.owner, inbox)
    )
    if delegation == null:
      throw NoValidChainError(capability)
    chain.append(delegation.cid)
    current = getParentCapability(delegation)

  return chain

function canReach(from, to, inbox) -> Boolean:
  // BFS to find delegation path from 'from' to 'to'
  visited = Set()
  queue = [from]
  while not queue.isEmpty():
    current = queue.pop()
    if current == to:
      return true
    visited.add(current)
    for d in inbox.where(delegatee == current):
      if d.delegator not in visited:
        queue.push(d.delegator)
  return false
```

## Examples

### Example 1: User Grants App Read Access to Photos

A user connects their wallet and grants a photo editing application read access to their photos.

```pseudocode
// Application initialization
app = TinyCloud({
  nodeUrl: "https://node.tinycloud.xyz",
  authorization: WebUserAuthorization()
})

// User connects wallet (triggers wallet popup)
userDID = await app.connect()

// Application requests photo read access
// This triggers a capability approval prompt
photoReadCap = Capability.kv()
  .read()
  .list()
  .atPath("/photos")

result = await app.requestCapability(photoReadCap)

if result.success:
  // User approved - app can now read photos
  photos = await app.storage.list("/photos")
  for photo in photos:
    data = await app.storage.get(photo)
    displayPhoto(data)
else:
  showMessage("Photo access is required to use this feature")
```

### Example 2: App Re-delegates to AI Agent with Time Limit

An application delegates a subset of its capabilities to an AI agent for a limited time.

```pseudocode
// App has received delegation from user
userDelegation = app.inbox.get(delegationId)

// Create time-limited delegation for AI agent
agentDID = "did:key:z6MkAgent..."

result = await app.redelegate(userDelegation)
  .to(agentDID)
  .attenuate({ path: "/photos/to-edit" })  // Narrower than original
  .expiring("30m")                          // 30 minute limit
  .withFacts({
    purpose: "automated photo enhancement",
    model: "enhance-v2"
  })
  .send()

if result.success:
  // AI agent can now access photos for 30 minutes
  notifyAgent(result.delegation)
```

### Example 3: Node Service Auto-signs for Backend Operations

A backend service operates with pre-authorized capabilities, automatically signing all requests.

```pseudocode
// Backend service initialization
service = TinyCloud({
  nodeUrl: "https://node.tinycloud.xyz",
  authorization: NodeUserAuthorization({
    strategy: "auto-sign",
    signer: loadPrivateKey(env.SERVICE_KEY_PATH)
  })
})

// Service DID derived from the private key
serviceDID = service.getUserDID()

// Receive delegation from space owner (via out-of-band process)
// The delegation is stored in the service's inbox

// All operations are automatically signed
function processUserUpload(userId, data):
  path = "/uploads/" + userId + "/" + generateId()

  // No prompts - auto-sign handles authorization
  result = service.storage.put(path, data)

  if result.success:
    return { path: path, cid: result.cid }
  else:
    logError("Upload failed:", result.reason)
    throw ServiceError(result.reason)
```

### Example 4: CLI Prompts User for Permission

A command-line tool prompts the user interactively when capabilities are needed.

```pseudocode
// CLI initialization with event emitter
emitter = EventEmitter()

cli = TinyCloud({
  nodeUrl: "https://node.tinycloud.xyz",
  authorization: NodeUserAuthorization({
    strategy: "event-emitter",
    emitter: emitter
  })
})

// Set up prompt handling
emitter.on("capability-request", function(request) {
  print("\nCapability Request:")
  print("  Action: " + request.capability.ability)
  print("  Resource: " + request.capability.resource)
  print("  Requested by: " + request.requestor)
  print("")

  response = prompt("Allow this request? (y/n): ")

  if response == "y":
    emitter.emit("capability-response", {
      requestId: request.id,
      approved: true
    })
  else:
    emitter.emit("capability-response", {
      requestId: request.id,
      approved: false,
      reason: "User declined"
    })
})

// CLI command execution
function uploadCommand(filePath, destination):
  data = readFile(filePath)

  // This may trigger the capability-request event
  result = cli.storage.put(destination, data)

  if result.success:
    print("Uploaded successfully: " + result.cid)
  else:
    print("Upload failed: " + result.reason)
```

## References

- Appendix A: Background Technologies (UCAN fundamentals)
- Appendix C: IPLD Schema Definitions (Delegation and Invocation structures)
- Appendix E: SIWE ReCaps Example (Capability encoding)
- Appendix G: Package Architecture (SDK implementation details)
