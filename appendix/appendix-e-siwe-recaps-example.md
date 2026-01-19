# Appendix E: SIWE ReCaps Example

The following shows a Sign-In with Ethereum message with ReCaps capability delegation:

```json
{
  "domain": "demo.tinycloud.xyz",
  "address": "0x6a12c8594c5C850d57612CA58810ABb8aeBbC04B",
  "statement": "I further authorize the stated URI to perform the following actions on my behalf: (1) 'tinycloud.kv': 'get', 'put' for 'tinycloud:pkh:eip155:1:0x6a12...C04B:default/kv/demo-app'.",
  "uri": "did:key:z6MksDuzXANBQ17iUHQ9M1yhHaUbiadAeHRiQdRMfymNgSmk",
  "version": "1",
  "chainId": 1,
  "nonce": "a6uoSZI5TfxzI4IFf",
  "issuedAt": "2025-12-04T13:21:48.263Z",
  "resources": [
    "urn:recap:eyJhdHQiOnsi..."
  ]
}
```

The `resources` array contains a base64-encoded ReCaps object specifying the delegated capabilities.

## TinyCloud URI Examples

**Key-Value Store Resource:**
```
tinycloud:pkh:eip155:1:0x6a12c8594c5C850d57612CA58810ABb8aeBbC04B:default/kv/photos/vacation.jpg
```

**SQL Database Resource:**
```
tinycloud:pkh:eip155:1:0x6a12c8594c5C850d57612CA58810ABb8aeBbC04B:default/sql/myapp.db
```

**Capabilities Resource:**
```
tinycloud:pkh:eip155:1:0x6a12c8594c5C850d57612CA58810ABb8aeBbC04B:default/capabilities/all
```

**Host Resource:**
```
tinycloud:key:z6MkiTBz1ymuepAQ4HEHYSF1H8quG5GLVVQR3djdX3mDooWp:default/hosts/*
```

## References

- W. Entriken et al., "EIP-4361: Sign-In with Ethereum," Ethereum Improvement Proposals, October 2021. [Online]. Available: https://eips.ethereum.org/EIPS/eip-4361
- O. Terbu et al., "EIP-5573: ReCaps - SIWE Capability Delegation," Ethereum Improvement Proposals, 2022. [Online]. Available: https://eips.ethereum.org/EIPS/eip-5573
