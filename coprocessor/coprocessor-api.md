# `Coprocessor` API

## Scope

Define the internal API, chain-ingestion rules, execution flow, and
`MPC`-facing request formats used by the coprocessor.

## Serialization

JSON is the transport format for internal API requests and responses.

Binary values are encoded as lowercase `0x`-prefixed hex strings.

Timestamps are Unix seconds.

## Core Types

```rust
pub type Address = [u8; 20];
pub type Bytes32 = [u8; 32];
pub type UnixSeconds = u64;

pub struct RequestId(pub Bytes32);
pub struct HandleId(pub Bytes32);
pub struct EnclaveMeasurement(pub Bytes32);

pub struct X25519PublicKey(pub [u8; 32]);
pub struct Attestation(pub Vec<u8>);

pub struct ChainEventRef {
    pub chain_id: u64,
    pub block_number: u64,
    pub block_hash: Bytes32,
    pub tx_hash: Bytes32,
    pub log_index: u32,
}
```

## Handle Resolution Status

```rust
pub enum HandleResolutionStatus {
    Pending,
    Ready,
    Failed,
}
```

A handle is:

- `Pending` when its dependencies are not yet available or execution is still
  running
- `Ready` when the coprocessor has produced `SystemCiphertextV1` for the
  handle
- `Failed` when execution or result packaging could not complete

## Chain Ingestion

The coprocessor host monitors the `symVM` event stream and the chain metadata
needed to reconstruct handle lineage.

Ingestion rules:

- use the `safe` chain view by default
- identify each consumed event by `(chain_id, block_hash, tx_hash, log_index)`
- discard derived state from orphaned blocks
- reconstruct handle dependencies from `symVM` events and contract metadata
- create pending handle state for newly observed derived handles
- schedule execution only after the required inputs for a handle are available

A deployment may define a stricter chain view such as `finalized`.

## Host And Enclave Split

The coprocessor host is responsible for:

- chain monitoring
- handle and dependency tracking
- execution task scheduling
- calls to `MPC`
- serving the internal handle-resolution API

The enclave is responsible for:

- generating an ephemeral enclave keypair
- producing attestation material for that key
- decrypting `EnclaveCiphertextV1` inputs
- evaluating symbolic operations over plaintext private values
- packaging resolved outputs as `SystemCiphertextV1`

## Execution Flow

The coprocessor resolves a handle in the following order:

1. ingest the `symVM` handle and its dependency metadata
2. create or update pending handle state
3. once dependencies are available, create an execution task
4. generate an enclave keypair and attestation
5. ask `MPC` to transform each required input `SystemCiphertextV1` payload to
   `EnclaveCiphertextV1`
6. decrypt the enclave-targeted ciphertexts inside the enclave
7. evaluate the symbolic operation inside the enclave
8. package the result as `SystemCiphertextV1` using the public configuration
   published by `MPC`
9. bind the result ciphertext and execution receipt to the handle

## Internal API

### `POST /internal/v1/handles/resolve`

Begins resolution for a handle or returns its current state.

```rust
pub struct ResolveHandleRequest {
    pub request_id: RequestId,
    pub chain_id: u64,
    pub contract: Address,
    pub handle_id: HandleId,
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

Repeated requests for the same `(chain_id, contract, handle_id)` attach to the
current resolution state for that handle.

### `GET /internal/v1/contracts/{contract}/handles/{handle_id}`

Returns the current state of a handle known to the coprocessor.

```rust
pub enum GetHandleResponse {
    Pending {
        handle_id: HandleId,
        status: HandleResolutionStatus,
    },
    Ready {
        handle_id: HandleId,
        status: HandleResolutionStatus,
        system_ciphertext: SystemCiphertextV1,
        receipt: Vec<u8>,
    },
    Failed {
        handle_id: HandleId,
        status: HandleResolutionStatus,
        error: String,
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
- `404` for unknown handles
- `409` for conflicting handle state
- `422` for invalid handle lineage or unresolved dependencies
- `503` for backend availability failures

## `Coprocessor` -> `MPC`

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

The coprocessor requests one `to-enclave` transformation for each input
`SystemCiphertextV1` required by the execution task.

## Ciphertext Types

The coprocessor treats these payloads as types defined by the `MPC`
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

pub struct EnclaveCiphertextV1 {
    pub key_id: Bytes32,
    pub enc: Vec<u8>,
    pub ciphertext: Vec<u8>,
    pub aad: Vec<u8>,
}
```
