# `MPC` Specification

## Goal

Define the role of `MPC` in the `Symbolic Execution` stack and the API surface
through which the `Coordinator` and the coprocessor request key operations.

`MPC` is scoped to key custody and threshold authorization. It is not the
general private execution layer.

## Working Model

`MPC` is the threshold-cryptography layer responsible for key custody,
authorized ciphertext transformation, re-encryption, and resilience.

Its API is used by the `Coordinator` and the coprocessor.

The transport protocol is HTTP.

Architecture:

- the coprocessor performs private execution
- the `Coordinator` handles public request orchestration
- `MPC` controls sensitive key operations

Enclave authorization uses the following model:

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

The base spec has no direct `MPC` to contract interface. A higher-level ERC
that needs a contract-visible completion path defines that packaging
separately.

## HTTP API

Transport rules:

- use HTTPS for every deployment-facing endpoint
- use JSON for control-plane fields and identifiers
- use canonical CBOR for cryptographic payloads
- if an endpoint stays JSON, carry binary payloads as base64url-encoded CBOR
- version every endpoint and ciphertext envelope explicitly

Endpoints:

- `GET /v1/config` for the current key configuration and supported suites
- `PUT /v1/readers/{reader_id}` to register or rotate a reader public key
- `POST /v1/operations/to-enclave` for coprocessor-driven transform to an
  attested enclave key
- `POST /v1/operations/to-reader` for reader-targeted disclosure
- `POST /v1/operations/sign` for threshold signatures or equivalent artifacts

Every `MPC` request is bound to:

- `version`
- `chain_id`
- `domain_id`
- `request_id`
- `key_id`

Requests from the coprocessor also carry an attestation or receipt
digest that binds the key operation to a specific execution result and enclave
public key.

For `POST /v1/operations/to-enclave`, the coprocessor provides at least:

- `enclave_pubkey`
- `attestation`
- `measurement`
- `request_id`
- `handle_id`
- `chain_id`

## Ciphertext Formats

This spec standardizes three envelope types plus one shared inner
plaintext encoding.

### Inner Plaintext Encoding

Encode plaintext payloads as canonical CBOR.

For the initial typed handle set:

- `suint256` values are encoded as 32-byte big-endian byte strings
- `sbool` values are encoded as a single byte, `0x00` or `0x01`

The AEAD additional authenticated data (`aad`) binds the ciphertext to
protocol context such as:

- `chain_id`
- `domain_id`
- `contract`
- `handle_id` or `request_id`
- `type_tag`
- `key_id`

### `SystemCiphertextV1`

Use this envelope for values held under system control and later transformed by
`MPC` for either enclave execution or reader disclosure.

Construction:

- generate a random 32-byte data-encryption key
- encrypt the canonical-CBOR plaintext with `AES-256-GCM`
- wrap the data-encryption key to the current `MPC` public configuration using
  `HPKE` with `X25519`, `HKDF-SHA256`, and `AES-256-GCM`
- carry `key_id`, `enc`, `wrapped_key`, `nonce`, `ciphertext`, and `aad`

### `EnclaveCiphertextV1`

Use this envelope when `MPC` authorizes execution and transforms a
`SystemCiphertextV1` payload to the enclave's attested public key.

Construction:

- use `HPKE` directly to the enclave ephemeral public key
- use `X25519`, `HKDF-SHA256`, and `AES-256-GCM`
- carry `key_id`, `enc`, `ciphertext`, and `aad`
- bind `request_id`, `handle_id`, `chain_id`, and the enclave attestation
  digest in `aad`

### `ReaderCiphertextV1`

Use this envelope for authorized disclosure from `MPC` to a reader key managed
by `sym-client`, typically delivered through the `Coordinator`.

Reader keys are dedicated encryption keys managed by `sym-client`, not the
user's Ethereum signing key.

Construction:

- use `HPKE` directly to the reader public key
- use `X25519`, `HKDF-SHA256`, and `AES-256-GCM`
- carry `key_id`, `enc`, `ciphertext`, and `aad`
- bind `request_id`, `reader_id`, `handle_id`, and `chain_id` in `aad`
