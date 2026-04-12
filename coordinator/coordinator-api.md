# `Coordinator` API

## Scope

Define the public HTTP API and disclosure authorization rules used by the
`Coordinator`.

The coordinator is the public API boundary between `sym-client` and the
backend services. Backend-specific interfaces are defined in the coprocessor
and `MPC` specs.

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

`reader_id` is `keccak256(reader_pubkey)`.

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

## Public API

### `GET /v1/config`

Returns the public configuration needed by `sym-client` for encryption and
coordinator requests.

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

pub struct RegisterReader {
    pub controller: Address,
    pub reader_id: ReaderId,
    pub nonce: Nonce,
    pub expiry: UnixSeconds,
}
```

The coordinator verifies that `reader_id == keccak256(reader_pubkey)`.

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

pub struct DisclosureRequest {
    pub contract: Address,
    pub handle_id: HandleId,
    pub reader_id: ReaderId,
    pub nonce: Nonce,
    pub expiry: UnixSeconds,
}

pub enum PostDisclosureResponse {
    Ready {
        request_id: RequestId,
        status: DisclosureStatus,
        ciphertext: Vec<u8>,
    },
    Pending {
        request_id: RequestId,
        status: DisclosureStatus,
    },
}
```

The `Ready` response returns `ReaderCiphertextV1`, encoded according to the
`MPC` spec.

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
        ciphertext: Vec<u8>,
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

The `Ready` response returns `ReaderCiphertextV1`, encoded according to the
`MPC` spec.

## Authorization

Disclosure authorization uses the following algorithm:

1. parse and validate the request body
2. verify `expiry >= now`
3. verify the nonce has not been used for the controlling account
4. resolve the controlling account from chain state
5. verify the EIP-712 signature against that controlling account
6. verify that `reader_id` is registered to the same controlling account

The controlling account is resolved by the higher-level standard.

Replay protection is enforced by `(controller, nonce)`.

## Disclosure Processing

The coordinator processes `POST /v1/disclosures` in the following order:

1. run the authorization algorithm
2. ask the coprocessor for `SystemCiphertextV1` if the handle is not already ready
3. ask `MPC` to transform that ciphertext to `ReaderCiphertextV1`
4. return `200` if the ciphertext is ready, otherwise return `202` with a
   `request_id`

`SystemCiphertextV1` and `ReaderCiphertextV1` are defined in
[`../mpc/mpc-api.md`](../mpc/mpc-api.md).

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
