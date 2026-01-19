# Appendix B: TinyCloud URI ABNF Grammar

The following ABNF grammar (RFC 5234) formally defines the syntax of string primitive types used in TinyCloud DAG nodes.

```abnf
;; ============================================================
;; TinyCloud String Primitives - ABNF Grammar (RFC 5234)
;; ============================================================

;; ------------------------------------------------------------
;; DID (Decentralized Identifier)
;; Reference: W3C DID Core Specification
;; ------------------------------------------------------------

DID                = "did:" method-name ":" method-specific-id
method-name        = 1*method-char
method-char        = %x61-7A / DIGIT  ; lowercase letter or digit
method-specific-id = *idchar *(":" *idchar)
idchar             = ALPHA / DIGIT / "." / "-" / "_" / pct-encoded
pct-encoded        = "%" HEXDIG HEXDIG

;; Common DID method patterns used in TinyCloud:
;;   did:key:z6Mk...     (Ed25519 public key)
;;   did:pkh:eip155:1:0x... (Ethereum address)

;; ------------------------------------------------------------
;; ResourceURI (TinyCloud Autonomic Space URI)
;; ------------------------------------------------------------

ResourceURI        = "tinycloud:" did-suffix ":" space "/" service-path
did-suffix         = method-name ":" method-specific-id
space              = 1*space-char
space-char         = ALPHA / DIGIT / "-" / "_"
service-path       = service ["/" path]
service            = 1*service-char
service-char       = ALPHA / DIGIT / "-" / "_"
path               = path-segment *("/" path-segment)
path-segment       = *pchar
pchar              = unreserved / pct-encoded / ":" / "@"
unreserved         = ALPHA / DIGIT / "-" / "." / "_" / "~"

;; Wildcard support for capability patterns:
path-pattern       = path-segment *("/" path-segment) ["/" "*"]

;; Examples:
;;   tinycloud:pkh:eip155:1:0x6a12...:default/kv/photos/vacation.jpg
;;   tinycloud:key:z6Mk...:work/capabilities/all
;;   tinycloud:key:z6Mk...:default/hosts/*

;; ------------------------------------------------------------
;; Ability (Capability Action Identifier)
;; ------------------------------------------------------------

Ability            = ability-namespace "/" ability-action
ability-namespace  = 1*ability-ns-char *("." 1*ability-ns-char)
ability-ns-char    = ALPHA / DIGIT / "-" / "_"
ability-action     = 1*ability-act-char
ability-act-char   = ALPHA / DIGIT / "-" / "_"

;; Wildcard for capability matching:
ability-pattern    = ability-namespace "/" ("*" / ability-action)

;; Standard TinyCloud abilities:
;;   tinycloud.kv/get       - Read from key-value store
;;   tinycloud.kv/put       - Write to key-value store
;;   tinycloud.kv/del       - Delete from key-value store
;;   tinycloud.kv/list      - List keys
;;   tinycloud.kv/metadata  - Read metadata
;;   tinycloud.space/host   - Host capability
;;
;; SQL Database abilities:
;;   tinycloud.sql/read     - Full read access (any SELECT)
;;   tinycloud.sql/write    - Full write access (INSERT, UPDATE, DELETE)
;;   tinycloud.sql/admin    - Schema changes (CREATE, ALTER, DROP)
;;   tinycloud.sql/select   - SELECT with table/column caveats
;;   tinycloud.sql/insert   - INSERT with table caveats
;;   tinycloud.sql/update   - UPDATE with table/column caveats
;;   tinycloud.sql/delete   - DELETE with table caveats
;;   tinycloud.sql/execute  - Execute specific prepared statements

;; ------------------------------------------------------------
;; Path (Key-Value Store Path)
;; ------------------------------------------------------------

Path               = ["/" ] path-segment *("/" path-segment)
path-segment       = 1*path-char
path-char          = ALPHA / DIGIT / "-" / "_" / "." / "~"
                   / pct-encoded

;; Path pattern for capability matching:
path-match-pattern = path-segment *("/" path-segment) ["/" "*"]

;; Examples:
;;   photos/vacation.jpg
;;   data/users/alice/profile.json
;;   config

;; ------------------------------------------------------------
;; Core Rules (from RFC 5234 Appendix B.1)
;; ------------------------------------------------------------

ALPHA              = %x41-5A / %x61-7A   ; A-Z / a-z
DIGIT              = %x30-39             ; 0-9
HEXDIG             = DIGIT / "A" / "B" / "C" / "D" / "E" / "F"
                   / "a" / "b" / "c" / "d" / "e" / "f"
```
