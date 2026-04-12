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
pub type Address = [u8; 20];
pub type Bytes32 = [u8; 32];

pub struct DomainId(pub Bytes32);
pub struct KeyId(pub Bytes32);
pub struct RequestId(pub Bytes32);
pub struct ReaderId(pub Bytes32);
pub struct HandleId(pub Bytes32);
pub struct EnclaveMeasurement(pub Bytes32);
pub struct AttestationDigest(pub Bytes32);

pub struct X25519PublicKey(pub [u8; 32]);
pub struct Attestation(pub Vec<u8>);
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
3. parse `system_ciphertext.aad` as `SystemInputAadV1` or `SystemHandleAadV1`
4. verify `request.chain_id == system_ciphertext.aad.chain_id`
5. if the source `aad` is `SystemHandleAadV1`, verify
   `request.handle_id == system_ciphertext.aad.handle_id`
6. verify the attestation binds `enclave_pubkey` to `measurement`
7. verify `measurement` matches the approved enclave measurement
8. construct `EnclaveAadV1` from the request and the source system `aad`
9. return `EnclaveCiphertextV1`

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
4. parse `system_ciphertext.aad` as `SystemHandleAadV1`
5. verify `request.chain_id == system_ciphertext.aad.chain_id`
6. verify `request.handle_id == system_ciphertext.aad.handle_id`
7. construct `ReaderAadV1` from the request and the source system `aad`
8. return `ReaderCiphertextV1`

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

### Canonical `aad` Encoding

`aad` is the canonical CBOR encoding of a fixed-length array.

It is not encoded as a map.

Encoding rules:

- every `aad` payload starts with `version`
- `version` is encoded as a CBOR unsigned integer
- `kind` is encoded as a CBOR unsigned integer
- `chain_id` is encoded as a CBOR unsigned integer
- `Address` is encoded as a 20-byte CBOR byte string
- `Bytes32`, `DomainId`, `KeyId`, `RequestId`, `ReaderId`, `HandleId`, and
  `AttestationDigest` are encoded as 32-byte CBOR byte strings
- `type_tag` is encoded as a CBOR text string
- array element order is fixed by the schema

```rust
pub enum AadKind {
    SystemInput = 1,
    SystemHandle = 2,
    Enclave = 3,
    Reader = 4,
}

pub struct SystemInputAadV1 {
    pub version: u8,
    pub kind: AadKind,
    pub chain_id: u64,
    pub domain_id: DomainId,
    pub contract: Address,
    pub type_tag: String,
    pub key_id: KeyId,
}

pub struct SystemHandleAadV1 {
    pub version: u8,
    pub kind: AadKind,
    pub chain_id: u64,
    pub domain_id: DomainId,
    pub handle_id: HandleId,
    pub type_tag: String,
    pub key_id: KeyId,
}

pub struct EnclaveAadV1 {
    pub version: u8,
    pub kind: AadKind,
    pub chain_id: u64,
    pub domain_id: DomainId,
    pub request_id: RequestId,
    pub handle_id: HandleId,
    pub type_tag: String,
    pub attestation_digest: AttestationDigest,
    pub key_id: KeyId,
}

pub struct ReaderAadV1 {
    pub version: u8,
    pub kind: AadKind,
    pub chain_id: u64,
    pub domain_id: DomainId,
    pub request_id: RequestId,
    pub handle_id: HandleId,
    pub reader_id: ReaderId,
    pub type_tag: String,
    pub key_id: KeyId,
}
```

The encoded arrays are:

- `SystemInputAadV1`:
  `[version, kind, chain_id, domain_id, contract, type_tag, key_id]`
- `SystemHandleAadV1`:
  `[version, kind, chain_id, domain_id, handle_id, type_tag, key_id]`
- `EnclaveAadV1`:
  `[version, kind, chain_id, domain_id, request_id, handle_id, type_tag, attestation_digest, key_id]`
- `ReaderAadV1`:
  `[version, kind, chain_id, domain_id, request_id, handle_id, reader_id, type_tag, key_id]`

Derivation rules:

- `EnclaveAadV1` copies `chain_id`, `domain_id`, `type_tag`, and `key_id`
  from the source `SystemInputAadV1` or `SystemHandleAadV1`
- `ReaderAadV1` copies `chain_id`, `domain_id`, `type_tag`, and `key_id`
  from the source `SystemHandleAadV1`

`attestation_digest` is defined as:

```rust
pub fn attestation_digest(attestation: Attestation) -> AttestationDigest {
    AttestationDigest(keccak256(attestation.0))
}
```

### Inner Plaintext Encoding

Plaintext payloads are encoded as canonical CBOR.

For the initial typed handle set:

- `suint256` values are encoded as 32-byte big-endian byte strings
- `sbool` values are encoded as a single byte, `0x00` or `0x01`

The AEAD additional authenticated data (`aad`) uses the canonical encoding
defined above.

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
- encode `aad` as `SystemInputAadV1` for client-encrypted inputs
- encode `aad` as `SystemHandleAadV1` for handle-bound system values

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
- encode `aad` as `EnclaveAadV1`

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
- encode `aad` as `ReaderAadV1`
