# `MPC` Specification

## Goal

Define the role of `MPC` in the `Symbolic Execution` stack and the API surface
through which the `Coordinator` and the coprocessor request key operations.

This draft keeps `MPC` narrowly scoped to key custody and threshold
authorization. It does not treat `MPC` as the general private execution layer.

## Working Model

`MPC` is the threshold-cryptography layer responsible for key custody,
authorized ciphertext transformation, re-encryption, and resilience.

Its API is used by the `Coordinator` and the coprocessor.

The transport protocol is HTTP.

In the current architecture direction:

- the coprocessor performs private execution
- the `Coordinator` handles public request orchestration
- `MPC` controls sensitive key operations

This separates computation from unilateral key power.

For v1, enclave authorization should also stay simple:

- each deployment approves a single enclave measurement
- `MPC` verifies that the attestation binds the enclave public key to that
  approved measurement

## Minimum Responsibilities

- hold key shares rather than a single decryption key
- expose an internal HTTP API for threshold key operations
- participate in authorized ciphertext-transformation and re-encryption flows
- produce threshold artifacts needed for disclosure or verification
- define stable ciphertext envelopes for system-held values, enclave-targeted
  outputs, and reader-targeted outputs
- never release raw decryption material to other services
- avoid giving any single operator unilateral control over private outputs

## Interfaces

### `Coordinator` -> `MPC`

The coordinator uses the `MPC` API to:

- fetch the current public-key and parameter bundle
- register or rotate reader keys
- request reader-targeted re-encryption
- receive reader-targeted ciphertext packages

### Coprocessor -> `MPC`

The coprocessor uses the `MPC` API after it has produced the required
execution context and attestation material.

The minimal request classes are:

- authorized transformation of a system-held ciphertext to an attested enclave
  public key
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
- `POST /v1/operations/to-enclave` for coprocessor-driven transform to an
  attested enclave key
- `POST /v1/operations/to-reader` for reader-targeted disclosure
- `POST /v1/operations/sign` for threshold signatures or equivalent artifacts

Every `MPC` request should be bound to:

- `version`
- `chain_id`
- `domain_id`
- `request_id`
- `purpose`
- `key_id`

Requests from the coprocessor should also carry an attestation or receipt
digest that binds the key operation to a specific execution result and enclave
public key.

For `POST /v1/operations/to-enclave`, the coprocessor should provide at least:

- `enclave_pubkey`
- `attestation`
- `measurement`
- `request_id`
- `handle_id`
- `chain_id`
- `purpose`

## Ciphertext Format Recommendations

The first draft should standardize three envelope types plus one shared inner
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

Use this envelope for values held under system control and later transformed by
`MPC` for either enclave execution or reader disclosure.

Recommended construction:

- generate a random 32-byte data-encryption key
- encrypt the canonical-CBOR plaintext with `AES-256-GCM`
- wrap the data-encryption key to the current `MPC` public configuration using
  `HPKE` with `X25519`, `HKDF-SHA256`, and `AES-256-GCM`
- carry `key_id`, `enc`, `wrapped_key`, `nonce`, `ciphertext`, and `aad`

This keeps payload encryption simple while allowing key rotation and future
re-encryption flows.

### `EnclaveCiphertextV1`

Use this envelope when `MPC` authorizes execution and transforms a
`SystemCiphertextV1` payload to the enclave's attested public key.

Recommended construction:

- use `HPKE` directly to the enclave ephemeral public key
- use `X25519`, `HKDF-SHA256`, and `AES-256-GCM`
- carry `key_id`, `enc`, `ciphertext`, and `aad`
- bind `request_id`, `handle_id`, `chain_id`, `purpose`, and the enclave
  attestation digest in `aad`

This keeps plaintext inside the enclave boundary while avoiding export of raw
decryption material.

### `ReaderCiphertextV1`

Use this envelope for authorized disclosure from `MPC` to a reader key managed
by `sym-client`, typically delivered through the `Coordinator`.

Reader keys are dedicated encryption keys managed by `sym-client`, not the
user's Ethereum signing key.

Recommended construction:

- use `HPKE` directly to the reader public key
- use `X25519`, `HKDF-SHA256`, and `AES-256-GCM`
- carry `key_id`, `enc`, `ciphertext`, and `aad`
- bind `request_id`, `reader_id`, `handle_id`, `chain_id`, and `purpose` in
  `aad`

This avoids exposing a reusable system decryption key to the reader path.
