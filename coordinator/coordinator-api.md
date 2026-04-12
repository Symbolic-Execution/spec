# `Coordinator` API

## Goal

Define the public HTTP API, wire types, and signed request formats used by
`sym-client` when it interacts with the `Coordinator`.

## Serialization

JSON is the transport format for public API requests and responses.

Binary values are encoded as lowercase `0x`-prefixed hex strings.

Timestamps are Unix seconds.

## Core Types

```rust
pub type Address = [u8; 20];
pub type Bytes32 = [u8; 32];
pub type UnixSeconds = u64;

#[derive(Clone, Copy, Debug, PartialEq, Eq, Hash)]
pub struct ReaderId(pub Bytes32);

#[derive(Clone, Copy, Debug, PartialEq, Eq, Hash)]
pub struct RequestId(pub Bytes32);

#[derive(Clone, Copy, Debug, PartialEq, Eq, Hash)]
pub struct HandleId(pub Bytes32);

#[derive(Clone, Copy, Debug, PartialEq, Eq, Hash)]
pub struct Nonce(pub Bytes32);

#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub struct X25519PublicKey(pub [u8; 32]);

#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub struct EthereumSignature(pub [u8; 65]);
```

## `reader_id`

`reader_id` is defined as:

```rust
pub fn reader_id(reader_pubkey: X25519PublicKey) -> ReaderId {
    ReaderId(keccak256(reader_pubkey.0))
}
```

The path parameter in `PUT /v1/readers/{reader_id}` must match the hash of the
supplied reader public key.

## Request Status

```rust
#[derive(Clone, Copy, Debug, PartialEq, Eq)]
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

## `GET /v1/config`

Returns the public configuration needed by `sym-client` for encryption and
signing.

```rust
#[derive(Clone, Debug, PartialEq, Eq)]
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

#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub enum ReaderKeyAlgorithm {
    X25519,
}

#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub enum CiphertextSuite {
    HpkeX25519HkdfSha256Aes256Gcm,
}

#[derive(Clone, Debug, PartialEq, Eq)]
pub struct Eip712Domain {
    pub name: String,
    pub version: String,
    pub chain_id: u64,
    pub salt: Bytes32,
}
```

## `PUT /v1/readers/{reader_id}`

Registers or rotates a reader key.

```rust
#[derive(Clone, Debug, PartialEq, Eq)]
pub struct PutReaderRequest {
    pub controller: Address,
    pub reader_pubkey: X25519PublicKey,
    pub nonce: Nonce,
    pub expiry: UnixSeconds,
    pub signature: EthereumSignature,
}

#[derive(Clone, Debug, PartialEq, Eq)]
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

## `POST /v1/disclosures`

Creates a disclosure request and either completes it immediately or returns a
pending request id.

```rust
#[derive(Clone, Debug, PartialEq, Eq)]
pub struct PostDisclosureRequest {
    pub contract: Address,
    pub handle_id: HandleId,
    pub reader_id: ReaderId,
    pub nonce: Nonce,
    pub expiry: UnixSeconds,
    pub signature: EthereumSignature,
}

#[derive(Clone, Debug, PartialEq, Eq)]
pub enum PostDisclosureResponse {
    Ready(ReadyDisclosureResponse),
    Pending(PendingDisclosureResponse),
}

#[derive(Clone, Debug, PartialEq, Eq)]
pub struct ReadyDisclosureResponse {
    pub request_id: RequestId,
    pub status: DisclosureStatus,
    pub ciphertext: ReaderCiphertextV1,
}

#[derive(Clone, Debug, PartialEq, Eq)]
pub struct PendingDisclosureResponse {
    pub request_id: RequestId,
    pub status: DisclosureStatus,
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
    pub name: &'static str,     // "SymbolicExecutionCoordinator"
    pub version: &'static str,  // "1"
    pub chain_id: u64,
    pub salt: Bytes32,          // domain_id
}
```

## `GET /v1/disclosures/{request_id}`

Returns the current state of a pending disclosure request.

```rust
#[derive(Clone, Debug, PartialEq, Eq)]
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
        error: CoordinatorErrorCode,
    },
    Expired {
        request_id: RequestId,
        status: DisclosureStatus,
    },
}
```

## Error Model

```rust
#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub enum CoordinatorErrorCode {
    InvalidSignature,
    ExpiredRequest,
    NonceAlreadyUsed,
    UnknownReader,
    ReaderIdMismatch,
    Unauthorized,
    HandleNotFound,
    ResultNotReady,
    BackendUnavailable,
    InvalidRequest,
}

#[derive(Clone, Debug, PartialEq, Eq)]
pub struct ErrorResponse {
    pub code: CoordinatorErrorCode,
    pub message: String,
}
```

HTTP status mapping:

- `400` for malformed requests and `InvalidRequest`
- `401` for `InvalidSignature`
- `403` for `Unauthorized`
- `404` for `HandleNotFound` and unknown `request_id`
- `409` for `NonceAlreadyUsed` and `ReaderIdMismatch`
- `410` for `ExpiredRequest`
- `422` for `UnknownReader`
- `503` for `BackendUnavailable`

## Ciphertext Type

The coordinator treats the reader-targeted payload as an opaque type defined by
the `MPC` spec.

```rust
#[derive(Clone, Debug, PartialEq, Eq)]
pub struct ReaderCiphertextV1 {
    pub key_id: Bytes32,
    pub enc: Vec<u8>,
    pub ciphertext: Vec<u8>,
    pub aad: Vec<u8>,
}
```
