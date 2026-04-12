# `MPC` API

## Scope

Define the internal HTTP API, authorization rules, and ciphertext formats used
by `MPC`.

## Serialization

The transport protocol is HTTPS.

HTTP endpoints use JSON for identifiers and control-plane fields.

Cryptographic payloads use canonical CBOR. When carried inside JSON, CBOR
payloads are encoded as base64url strings.

## Core Types

```rust
pub type Bytes32 = [u8; 32];

pub struct DomainId(pub Bytes32);
pub struct KeyId(pub Bytes32);
pub struct RequestId(pub Bytes32);
pub struct ReaderId(pub Bytes32);
pub struct HandleId(pub Bytes32);
pub struct EnclaveMeasurement(pub Bytes32);

pub struct X25519PublicKey(pub [u8; 32]);
pub struct Attestation(pub Vec<u8>);
pub struct MessageDigest(pub Bytes32);
```

## `reader_id`

`reader_id` is defined as:

```rust
pub fn reader_id(reader_pubkey: X25519PublicKey) -> ReaderId {
    ReaderId(keccak256(reader_pubkey.0))
}
```

The path parameter in `PUT /v1/readers/{reader_id}` matches the hash of the
supplied reader public key.

## Public Configuration

### `GET /v1/config`

Returns the public configuration used by `sym-client`, the coordinator, and
the coprocessor.

```rust
pub struct MpcConfigResponse {
    pub version: u16,
    pub chain_id: u64,
    pub domain_id: DomainId,
    pub key_id: KeyId,
    pub hpke_public_key: X25519PublicKey,
    pub reader_key_algorithm: ReaderKeyAlgorithm,
    pub ciphertext_suite: CiphertextSuite,
    pub approved_enclave_measurement: EnclaveMeasurement,
}

pub enum ReaderKeyAlgorithm {
    X25519,
}

pub enum CiphertextSuite {
    HpkeX25519HkdfSha256Aes256Gcm,
}
```

## Reader Registry

### `PUT /v1/readers/{reader_id}`

Registers a reader public key.

```rust
pub struct PutReaderRequest {
    pub reader_pubkey: X25519PublicKey,
}

pub struct PutReaderResponse {
    pub reader_id: ReaderId,
}
```

Registration is valid only if `reader_id == keccak256(reader_pubkey)`.

Registering the same `(reader_id, reader_pubkey)` pair again is idempotent.

Reader key rotation is represented by registering a new reader public key and
its derived `reader_id`.

## Ciphertext Transformation

### `POST /v1/operations/to-enclave`

Transforms `SystemCiphertextV1` to `EnclaveCiphertextV1` for an attested
enclave public key.

```rust
pub struct ToEnclaveRequest {
    pub request_id: RequestId,
    pub chain_id: u64,
    pub handle_id: HandleId,
    pub enclave_pubkey: X25519PublicKey,
    pub measurement: EnclaveMeasurement,
    pub attestation: Attestation,
    pub system_ciphertext: SystemCiphertextV1,
}

pub struct ToEnclaveResponse {
    pub ciphertext: EnclaveCiphertextV1,
}
```

Authorization rules:

1. parse and validate the request body
2. verify `system_ciphertext.key_id` is active
3. verify the attestation binds `enclave_pubkey` to `measurement`
4. verify `measurement` matches the approved enclave measurement
5. return `EnclaveCiphertextV1`

### `POST /v1/operations/to-reader`

Transforms `SystemCiphertextV1` to `ReaderCiphertextV1` for a registered
reader key.

```rust
pub struct ToReaderRequest {
    pub request_id: RequestId,
    pub chain_id: u64,
    pub handle_id: HandleId,
    pub reader_id: ReaderId,
    pub system_ciphertext: SystemCiphertextV1,
}

pub struct ToReaderResponse {
    pub ciphertext: ReaderCiphertextV1,
}
```

Authorization rules:

1. parse and validate the request body
2. resolve the registered reader public key for `reader_id`
3. verify `system_ciphertext.key_id` is active
4. return `ReaderCiphertextV1`

## Threshold Signatures

### `POST /v1/operations/sign`

Produces a threshold signature or equivalent authorization artifact.

```rust
pub struct SignRequest {
    pub request_id: RequestId,
    pub chain_id: u64,
    pub key_id: KeyId,
    pub message_digest: MessageDigest,
}

pub struct SignResponse {
    pub artifact: Vec<u8>,
}
```

The message format and artifact format are defined by the higher-level
standard that uses the signature.

## Error Responses

```rust
pub struct ErrorResponse {
    pub code: String,
    pub message: String,
}
```

HTTP status mapping:

- `400` for malformed requests
- `403` for authorization failures
- `404` for unknown reader ids and unknown key ids
- `409` for reader id mismatches
- `422` for invalid attestations or invalid ciphertext bindings
- `503` for backend availability failures

## Ciphertext Formats

### Inner Plaintext Encoding

Plaintext payloads are encoded as canonical CBOR.

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

`SystemCiphertextV1` is used for values held under system control.

```rust
pub struct SystemCiphertextV1 {
    pub key_id: KeyId,
    pub enc: Vec<u8>,
    pub wrapped_key: Vec<u8>,
    pub nonce: [u8; 12],
    pub ciphertext: Vec<u8>,
    pub aad: Vec<u8>,
}
```

Construction:

- generate a random 32-byte data-encryption key
- encrypt the canonical-CBOR plaintext with `AES-256-GCM`
- wrap the data-encryption key to the published `MPC` public configuration using
  `HPKE` with `X25519`, `HKDF-SHA256`, and `AES-256-GCM`

### `EnclaveCiphertextV1`

`EnclaveCiphertextV1` is used when `MPC` transforms a
`SystemCiphertextV1` payload to an enclave public key.

```rust
pub struct EnclaveCiphertextV1 {
    pub key_id: KeyId,
    pub enc: Vec<u8>,
    pub ciphertext: Vec<u8>,
    pub aad: Vec<u8>,
}
```

Construction:

- use `HPKE` directly to the enclave public key
- use `X25519`, `HKDF-SHA256`, and `AES-256-GCM`
- bind `request_id`, `handle_id`, `chain_id`, and the enclave attestation
  digest in `aad`

### `ReaderCiphertextV1`

`ReaderCiphertextV1` is used for reader-targeted disclosure.

```rust
pub struct ReaderCiphertextV1 {
    pub key_id: KeyId,
    pub enc: Vec<u8>,
    pub ciphertext: Vec<u8>,
    pub aad: Vec<u8>,
}
```

Construction:

- use `HPKE` directly to the reader public key
- use `X25519`, `HKDF-SHA256`, and `AES-256-GCM`
- bind `request_id`, `reader_id`, `handle_id`, and `chain_id` in `aad`
