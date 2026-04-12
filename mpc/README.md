# `MPC` Specification

## Goal

Define the role of `MPC` in the `Symbolic Execution` stack and the API surface
through which `sym-client` and the coprocessor request key operations.

This draft keeps `MPC` narrowly scoped to key custody and threshold
authorization. It does not treat `MPC` as the general private execution layer.

## Working Model

`MPC` is the threshold-cryptography layer responsible for key custody,
decryption authorization, re-encryption, and resilience.

Its API is the communication channel for both the coprocessor and
`sym-client`.

The transport protocol is HTTP.

In the current architecture direction:

- the coprocessor performs private execution
- `MPC` controls sensitive key operations
- `sym-client` uses `MPC` for reader-key and disclosure flows

This separates computation from unilateral key power.

## Minimum Responsibilities

- hold key shares rather than a single decryption key
- expose an HTTP API for threshold key operations
- participate in authorized decryption or re-encryption flows
- produce threshold artifacts needed for disclosure or verification
- define stable ciphertext envelopes for system-held values and reader-targeted
  outputs
- avoid giving any single operator unilateral control over private outputs

## Interfaces

### `sym-client` -> `MPC`

`sym-client` uses the `MPC` API to:

- fetch the current public-key and parameter bundle
- register or rotate reader keys
- submit encrypted private inputs when a flow requires `MPC` ingestion
- receive reader-targeted disclosure packages

### Coprocessor -> `MPC`

The coprocessor uses the `MPC` API after it has produced the required
execution context and attestation material.

The minimal request classes are:

- threshold decryption of system-held ciphertexts
- re-encryption to an authorized reader key
- threshold signatures or equivalent authorization artifacts

The base spec does not require a direct `MPC` to contract interface. If a
higher-level ERC needs a contract-visible completion path, it should specify
that packaging separately.

## HTTP API

The first draft should keep the transport simple:

- use HTTPS for every deployment-facing endpoint
- use JSON for control-plane fields and identifiers
- use canonical CBOR for cryptographic payloads
- if an endpoint stays JSON, carry binary payloads as base64url-encoded CBOR
- version every endpoint and ciphertext envelope explicitly

Recommended first endpoints:

- `GET /v1/config` for the current key configuration and supported suites
- `PUT /v1/readers/{reader_id}` to register or rotate a reader public key
- `POST /v1/inputs` to ingest a new encrypted private input
- `POST /v1/operations/decrypt` for coprocessor-driven threshold decryption
- `POST /v1/operations/reencrypt` for reader-targeted disclosure
- `POST /v1/operations/sign` for threshold signatures or equivalent artifacts
- `GET /v1/disclosures/{request_id}` to fetch a reader-targeted disclosure
  package

Every `MPC` request should be bound to:

- `version`
- `chain_id`
- `domain_id`
- `request_id`
- `purpose`
- `key_id`

Requests from the coprocessor should also carry an attestation or receipt
digest that binds the key operation to a specific execution result.

## Ciphertext Format Recommendations

The first draft should standardize two envelope types plus one shared inner
plaintext encoding.

### Inner Plaintext Encoding

Encode plaintext payloads as canonical CBOR.

For the first typed handle set:

- `suint256` values should be encoded as 32-byte big-endian byte strings
- `sbool` values should be encoded as a single byte, `0x00` or `0x01`

The AEAD additional authenticated data should bind the ciphertext to protocol
context such as:

- `chain_id`
- `domain_id`
- `contract`
- `handle_id` or `request_id`
- `purpose`
- `type_tag`
- `key_id`

### `SystemCiphertextV1`

Use this envelope for values held under system control and later consumed by
the coprocessor plus `MPC`.

Recommended construction:

- generate a random 32-byte data-encryption key
- encrypt the canonical-CBOR plaintext with `AES-256-GCM`
- wrap the data-encryption key to the current `MPC` public configuration using
  `HPKE` with `X25519`, `HKDF-SHA256`, and `AES-256-GCM`
- carry `key_id`, `enc`, `wrapped_key`, `nonce`, `ciphertext`, and `aad`

This keeps payload encryption simple while allowing key rotation and future
re-encryption flows.

### `ReaderCiphertextV1`

Use this envelope for authorized disclosure from `MPC` to a reader key managed
by `sym-client`.

Recommended construction:

- use `HPKE` directly to the reader public key
- use `X25519`, `HKDF-SHA256`, and `AES-256-GCM`
- carry `key_id`, `enc`, `ciphertext`, and `aad`
- bind `request_id`, `reader_id`, `handle_id`, `chain_id`, and `purpose` in
  `aad`

This avoids exposing a reusable system decryption key to the reader path.
