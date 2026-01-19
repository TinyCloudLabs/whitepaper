# Appendix A: Background Technologies

## A.1 Decentralized Identifiers (DIDs)

A Decentralized Identifier (DID) is a globally unique identifier that does not require a centralized registration authority [2]. DIDs are URIs that associate a DID subject with a DID document, enabling verifiable, decentralized digital identity.

**Example DID:**
```
did:key:z6MkiTBz1ymuepAQ4HEHYSF1H8quG5GLVVQR3djdX3mDooWp
```

DIDs provide the cryptographic foundation for TinyCloud, as each space is controlled by a DID whose private key holder has root authority over all resources.

**Reference:** W3C DID Core Specification [2]

## A.2 User-Controlled Authorization Networks (UCANs)

UCANs are a trustless, secure, local-first, user-originated authorization and revocation scheme [3]. They extend JSON Web Tokens (JWTs) to provide public-key verifiable, delegable, expressive capabilities.

Key UCAN concepts relevant to TinyCloud:

| Term | Definition |
|------|------------|
| **Capability** | An association of an ability to a resource: `resource × ability × caveats` |
| **Delegation** | The act of granting another principal the capability to use a resource |
| **Attenuation** | The process of constraining capabilities in a delegation chain |
| **Revocation** | Invalidating a UCAN after issuance |

**Example UCAN Capability:**
```json
{
  "tinycloud:pkh:eip155:1:0x6a12...C04B:default/kv/demo-app": {
    "tinycloud.kv/put": [{}],
    "tinycloud.kv/get": [{}]
  }
}
```

TinyCloud uses UCAN v0.10.0, which allows delegations to include multiple capabilities in a single token, minimizing user experience friction.

**Reference:** UCAN Specification v0.10.0 [3]

## A.3 Sign-In with Ethereum (SIWE) and ReCaps

For users with cryptocurrency wallets, TinyCloud supports Sign-In with Ethereum (SIWE) [4] with ReCaps extensions [5]. SIWE allows Ethereum account holders to sign messages for authentication, while ReCaps (EIP-5573) enables encoding UCAN-like capability semantics within SIWE messages.

UCANs and SIWE ReCaps are functionally equivalent—they represent the same authorization data in different serialization formats:

| Format | Use Case | Signature Type |
|--------|----------|----------------|
| UCAN | Native applications, server-to-server | EdDSA, ECDSA |
| SIWE + ReCaps | Ethereum wallet users | Ethereum personal_sign |

**References:** EIP-4361 (SIWE) [4], EIP-5573 (ReCaps) [5]

## A.4 Chain Agnostic CApability Object (CACAO)

CACAO (Chain Agnostic CApability Object), defined in CAIP-74 [11], provides the IPLD-based container format for storing signed capability delegations. A CACAO wraps the result of a SIWE or other chain-specific signing operation into a content-addressable, deterministically serializable object.

**CACAO Structure:**

| Component | Description |
|-----------|-------------|
| **Header (h)** | Identifies the payload format (e.g., `eip4361` or `caip122`) |
| **Payload (p)** | Contains domain, issuer DID, audience, timestamps, nonce, and resources |
| **Signature (s)** | The cryptographic signature and verification method |

CACAOs enable capability chaining—each CACAO can reference parent capabilities, forming a verifiable delegation tree. When serialized using DAG-CBOR, CACAOs are identified by their Content Identifier (CID), enabling content-addressed storage and retrieval.

**Example CACAO Payload:**
```json
{
  "h": { "t": "eip4361" },
  "p": {
    "domain": "demo.tinycloud.xyz",
    "iss": "did:pkh:eip155:1:0x6a12c8594c5C850d57612CA58810ABb8aeBbC04B",
    "aud": "did:key:z6MksDuzXANBQ17iUHQ9M1yhHaUbiadAeHRiQdRMfymNgSmk",
    "version": "1",
    "nonce": "a6uoSZI5TfxzI4IFf",
    "iat": "2025-12-04T13:21:48.263Z",
    "resources": ["urn:recap:eyJhdHQiOnsi..."]
  },
  "s": { "t": "eip191", "s": "0x..." }
}
```

**Reference:** CAIP-74 [11]

## A.5 The `did:pkh` Method

The `did:pkh` (DID Public Key Hash) method [12] enables blockchain addresses to function as Decentralized Identifiers without requiring on-chain registration. This method derives a DID directly from a blockchain account address.

**Format:**
```
did:pkh:<chain-namespace>:<chain-id>:<account-address>
```

**Example:**
```
did:pkh:eip155:1:0x6a12c8594c5C850d57612CA58810ABb8aeBbC04B
       └─────┘ └┘ └──────────────────────────────────────┘
       namespace chain-id        Ethereum address
```

| Component | Description |
|-----------|-------------|
| `eip155` | CAIP-2 namespace for EVM-compatible chains |
| `1` | Chain ID (1 = Ethereum mainnet) |
| `0x6a12...` | The Ethereum account address |

This DID method is particularly important for TinyCloud as it allows Ethereum users to control TinyCloud spaces using their existing wallet infrastructure without additional key management.

**Reference:** did:pkh Method Specification [12]

## A.6 Content Addressing and IPLD

TinyCloud uses content addressing through Content Identifiers (CIDs) to reference data immutably [6]. The InterPlanetary Linked Data (IPLD) model provides a canonical way to hash and link data structures into a directed acyclic graph (DAG).

CIDs ensure that:
- Data integrity can be verified without trusting the source
- References to data are stable across network partitions
- Deduplication occurs naturally for identical content

## References

- W3C, "Decentralized Identifiers (DIDs) v1.0," W3C Recommendation, July 2022. [Online]. Available: https://www.w3.org/TR/did-core/
- B. Zelenka, D. Holmgren, I. Gozalishvili, P. Krüger, "User Controlled Authorization Network (UCAN) Specification v0.10.0," 2024. [Online]. Available: https://github.com/ucan-wg/spec
- W. Entriken et al., "EIP-4361: Sign-In with Ethereum," Ethereum Improvement Proposals, October 2021. [Online]. Available: https://eips.ethereum.org/EIPS/eip-4361
- O. Terbu et al., "EIP-5573: ReCaps - SIWE Capability Delegation," Ethereum Improvement Proposals, 2022. [Online]. Available: https://eips.ethereum.org/EIPS/eip-5573
- Protocol Labs, "Content Identifiers (CIDs)," IPFS Documentation. [Online]. Available: https://docs.ipfs.io/concepts/content-addressing/
- S. Ukustov, Haardik, "CAIP-74: CACAO - Chain Agnostic CApability Object," Chain Agnostic Improvement Proposals, 2022. [Online]. Available: https://github.com/ChainAgnostic/CAIPs/blob/master/CAIPs/caip-74.md
- W3C Credentials Community Group, "did:pkh Method Specification," 2022. [Online]. Available: https://github.com/w3c-ccg/did-pkh/blob/main/did-pkh-method-draft.md
- ChainAgnostic, "CAIP-2: Blockchain ID Specification," Chain Agnostic Improvement Proposals. [Online]. Available: https://github.com/ChainAgnostic/CAIPs/blob/master/CAIPs/caip-2.md
