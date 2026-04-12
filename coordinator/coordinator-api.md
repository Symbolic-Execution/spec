# `Coordinator` API

## Scope

Define the public HTTP API, signed request formats, processing rules, and
backend interfaces used by the `Coordinator`.

## Serialization

JSON is the transport format for public API requests and responses.

Binary values are encoded as lowercase `0x`-prefixed hex strings.

Timestamps are Unix seconds.

## Core Types

```rust
pub type Address = [u8; 20];
pub type Bytes32 = [u8; 32];
pub type UnixSeconds = u64;

pub struct ReaderId(pub Bytes32);
pub struct RequestId(pub Bytes32);
pub struct HandleId(pub Bytes32);
pub struct Nonce(pub Bytes32);

pub struct X25519PublicKey(pub [u8; 32]);
pub struct EthereumSignature(pub [u8; 65]);
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

## Disclosure Status

```rust
pub enum DisclosureStatus {
    Pending,
    Ready,
    Failed,
    Expired,
}
```

State transitions:

- `Pending -> Ready`
- `Pending -> Failed`
- `Pending -> Expired`

`Ready`, `Failed`, and `Expired` are terminal states.

## Public API

### `GET /v1/config`

Returns the public configuration needed by `sym-client` for encryption and
signing.

```rust
pub struct CoordinatorConfigResponse {
    pub version: u16,
    pub chain_id: u64,
    pub domain_id: Bytes32,
    pub mpc_key_id: Bytes32,
    pub mpc_hpke_public_key: Bytes32,
    pub reader_key_algorithm: ReaderKeyAlgorithm,
    pub ciphertext_suite: CiphertextSuite,
    pub eip712_domain: Eip712Domain,
}

pub enum ReaderKeyAlgorithm {
    X25519,
}

pub enum CiphertextSuite {
    HpkeX25519HkdfSha256Aes256Gcm,
}

pub struct Eip712Domain {
    pub name: String,
    pub version: String,
    pub chain_id: u64,
    pub salt: Bytes32,
}
```

### `PUT /v1/readers/{reader_id}`

Registers or rotates a reader key.

```rust
pub struct PutReaderRequest {
    pub controller: Address,
    pub reader_pubkey: X25519PublicKey,
    pub nonce: Nonce,
    pub expiry: UnixSeconds,
    pub signature: EthereumSignature,
}

pub struct PutReaderResponse {
    pub reader_id: ReaderId,
    pub controller: Address,
    pub registered_at: UnixSeconds,
    pub expires_at: Option<UnixSeconds>,
}
```

The signature uses EIP-712 over the following typed message:

```rust
pub struct RegisterReader {
    pub controller: Address,
    pub reader_id: ReaderId,
    pub nonce: Nonce,
    pub expiry: UnixSeconds,
}
```

`reader_pubkey` is not signed directly. The signature binds `reader_id`, and
the coordinator verifies that `reader_id == keccak256(reader_pubkey)`.

### `POST /v1/disclosures`

Creates a disclosure request and either completes it immediately or returns a
pending request id.

```rust
pub struct PostDisclosureRequest {
    pub contract: Address,
    pub handle_id: HandleId,
    pub reader_id: ReaderId,
    pub nonce: Nonce,
    pub expiry: UnixSeconds,
    pub signature: EthereumSignature,
}

pub enum PostDisclosureResponse {
    Ready {
        request_id: RequestId,
        status: DisclosureStatus,
        ciphertext: ReaderCiphertextV1,
    },
    Pending {
        request_id: RequestId,
        status: DisclosureStatus,
    },
}
```

The signature uses EIP-712 over the following typed message:

```rust
pub struct DisclosureRequest {
    pub contract: Address,
    pub handle_id: HandleId,
    pub reader_id: ReaderId,
    pub nonce: Nonce,
    pub expiry: UnixSeconds,
}
```

The EIP-712 domain is:

```rust
pub struct CoordinatorEip712Domain {
    pub name: String,
    pub version: String,
    pub chain_id: u64,
    pub salt: Bytes32,
}
```

### `GET /v1/disclosures/{request_id}`

Returns the current state of a pending disclosure request.

```rust
pub enum GetDisclosureResponse {
    Pending {
        request_id: RequestId,
        status: DisclosureStatus,
    },
    Ready {
        request_id: RequestId,
        status: DisclosureStatus,
        ciphertext: ReaderCiphertextV1,
    },
    Failed {
        request_id: RequestId,
        status: DisclosureStatus,
        error: String,
    },
    Expired {
        request_id: RequestId,
        status: DisclosureStatus,
    },
}
```

## Error Responses

```rust
pub struct ErrorResponse {
    pub code: String,
    pub message: String,
}
```

HTTP status mapping:

- `400` for malformed requests
- `401` for invalid signatures
- `403` for authorization failures
- `404` for unknown handles and unknown request ids
- `409` for nonce reuse and reader id mismatches
- `410` for expired requests
- `422` for unknown readers
- `503` for backend availability failures

## Authorization

Disclosure authorization uses the following algorithm:

1. parse and validate the request body
2. verify `expiry >= now`
3. verify the nonce has not been used for the controlling account
4. resolve the controlling account from chain state
5. verify the EIP-712 signature against that controlling account
6. verify that `reader_id` is registered to the same controlling account

The controlling account is resolved by the higher-level standard.

## Disclosure Processing

The coordinator processes `POST /v1/disclosures` in the following order:

1. run the authorization algorithm
2. ask the coprocessor for the current `SystemCiphertextV1`
3. ask `MPC` to transform that ciphertext to `ReaderCiphertextV1`
4. return `200` if the ciphertext is ready, otherwise return `202` with a
   `request_id`

## `Coordinator` -> Coprocessor

```rust
pub struct ResolveHandleRequest {
    pub request_id: RequestId,
    pub chain_id: u64,
    pub contract: Address,
    pub handle_id: HandleId,
    pub purpose: String,
}

pub enum ResolveHandleResponse {
    Pending,
    Ready {
        system_ciphertext: SystemCiphertextV1,
        receipt: Vec<u8>,
    },
    Failed {
        reason: String,
    },
}
```

## `Coordinator` -> `MPC`

```rust
pub struct MpcRegisterReaderRequest {
    pub reader_id: ReaderId,
    pub reader_pubkey: X25519PublicKey,
}

pub struct MpcRegisterReaderResponse {
    pub reader_id: ReaderId,
}

pub struct ToReaderRequest {
    pub request_id: RequestId,
    pub chain_id: u64,
    pub handle_id: HandleId,
    pub reader_id: ReaderId,
    pub purpose: String,
    pub system_ciphertext: SystemCiphertextV1,
}

pub struct ToReaderResponse {
    pub ciphertext: ReaderCiphertextV1,
}
```

## Expiry And Replay Rules

- a disclosure request with `expiry < now` is rejected
- a `pending` disclosure request becomes `expired` when `expiry < now`
- replay protection is enforced by `(controller, nonce)`

## Ciphertext Types

The coordinator treats these payloads as opaque types defined by the `MPC`
specification.

```rust
pub struct SystemCiphertextV1 {
    pub key_id: Bytes32,
    pub enc: Vec<u8>,
    pub wrapped_key: Vec<u8>,
    pub nonce: [u8; 12],
    pub ciphertext: Vec<u8>,
    pub aad: Vec<u8>,
}

pub struct ReaderCiphertextV1 {
    pub key_id: Bytes32,
    pub enc: Vec<u8>,
    pub ciphertext: Vec<u8>,
    pub aad: Vec<u8>,
}
```
