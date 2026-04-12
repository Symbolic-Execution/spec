# `Coordinator` Runtime

## Goal

Define the backend interfaces, authorization steps, persistence model, and
request lifecycle used by the `Coordinator`.

## Controller Resolution

The `Coordinator` authorizes disclosure against the handle's controlling
account.

The controlling account is resolved by the higher-level standard.

```rust
pub trait ControllerResolver {
    fn controller_of(
        &self,
        contract: Address,
        handle_id: HandleId,
    ) -> Result<Address, ControllerResolutionError>;
}

#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub enum ControllerResolutionError {
    HandleNotFound,
    UnsupportedContract,
    ChainUnavailable,
}
```

## Authorization Algorithm

The `Coordinator` processes `POST /v1/disclosures` in the following order:

1. parse and validate the request body
2. verify `expiry >= now`
3. verify the nonce has not been used for the controlling account
4. resolve the controlling account from chain state
5. verify the EIP-712 signature against that controlling account
6. verify that `reader_id` is registered to the same controlling account
7. ask the coprocessor for the current `SystemCiphertextV1`
8. ask `MPC` to transform that ciphertext to `ReaderCiphertextV1`
9. return `200` if the ciphertext is ready, otherwise persist the request and
   return `202`

## Backend Interfaces

### `Coordinator` -> Coprocessor

```rust
pub trait CoprocessorClient {
    fn resolve_handle(
        &self,
        request: ResolveHandleRequest,
    ) -> Result<ResolveHandleResponse, CoprocessorError>;
}

#[derive(Clone, Debug, PartialEq, Eq)]
pub struct ResolveHandleRequest {
    pub request_id: RequestId,
    pub chain_id: u64,
    pub contract: Address,
    pub handle_id: HandleId,
    pub purpose: ResolvePurpose,
}

#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub enum ResolvePurpose {
    ReaderDisclosure,
}

#[derive(Clone, Debug, PartialEq, Eq)]
pub enum ResolveHandleResponse {
    Pending,
    Ready {
        system_ciphertext: SystemCiphertextV1,
        receipt: Vec<u8>,
    },
    Failed {
        code: CoprocessorErrorCode,
    },
}

#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub enum CoprocessorErrorCode {
    HandleNotFound,
    ResolutionFailed,
    AttestationInvalid,
    Unavailable,
}

#[derive(Clone, Debug, PartialEq, Eq)]
pub struct CoprocessorError {
    pub code: CoprocessorErrorCode,
}
```

### `Coordinator` -> `MPC`

```rust
pub trait MpcClient {
    fn register_reader(
        &self,
        request: RegisterReaderRequest,
    ) -> Result<RegisterReaderResult, MpcError>;

    fn to_reader(
        &self,
        request: ToReaderRequest,
    ) -> Result<ReaderCiphertextV1, MpcError>;
}

#[derive(Clone, Debug, PartialEq, Eq)]
pub struct RegisterReaderRequest {
    pub reader_id: ReaderId,
    pub reader_pubkey: X25519PublicKey,
}

#[derive(Clone, Debug, PartialEq, Eq)]
pub struct RegisterReaderResult {
    pub reader_id: ReaderId,
}

#[derive(Clone, Debug, PartialEq, Eq)]
pub struct ToReaderRequest {
    pub request_id: RequestId,
    pub chain_id: u64,
    pub handle_id: HandleId,
    pub reader_id: ReaderId,
    pub purpose: Bytes32,
    pub system_ciphertext: SystemCiphertextV1,
}

#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub enum MpcErrorCode {
    UnknownReader,
    Unauthorized,
    Unavailable,
}

#[derive(Clone, Debug, PartialEq, Eq)]
pub struct MpcError {
    pub code: MpcErrorCode,
}
```

## Persistence

The coordinator persists:

- registered reader keys
- used nonces
- disclosure request records
- final `ReaderCiphertextV1` for requests that reached `Ready`

```rust
pub trait CoordinatorStore {
    fn put_reader(&self, reader: ReaderRecord) -> Result<(), StoreError>;
    fn get_reader(&self, reader_id: ReaderId) -> Result<Option<ReaderRecord>, StoreError>;

    fn use_nonce(&self, controller: Address, nonce: Nonce, expiry: UnixSeconds)
        -> Result<(), StoreError>;
    fn has_nonce(&self, controller: Address, nonce: Nonce) -> Result<bool, StoreError>;

    fn put_disclosure(&self, record: DisclosureRecord) -> Result<(), StoreError>;
    fn get_disclosure(&self, request_id: RequestId)
        -> Result<Option<DisclosureRecord>, StoreError>;
}

#[derive(Clone, Debug, PartialEq, Eq)]
pub struct ReaderRecord {
    pub reader_id: ReaderId,
    pub controller: Address,
    pub reader_pubkey: X25519PublicKey,
    pub registered_at: UnixSeconds,
    pub expires_at: Option<UnixSeconds>,
}

#[derive(Clone, Debug, PartialEq, Eq)]
pub struct DisclosureRecord {
    pub request_id: RequestId,
    pub contract: Address,
    pub handle_id: HandleId,
    pub controller: Address,
    pub reader_id: ReaderId,
    pub nonce: Nonce,
    pub expiry: UnixSeconds,
    pub status: DisclosureStatus,
    pub ciphertext: Option<ReaderCiphertextV1>,
    pub error: Option<CoordinatorErrorCode>,
    pub created_at: UnixSeconds,
    pub updated_at: UnixSeconds,
}

#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub enum StoreError {
    Conflict,
    Unavailable,
}
```

## Expiry Rules

- a disclosure request with `expiry < now` is rejected
- a `pending` disclosure request becomes `expired` when `expiry < now`
- a stored `ready` ciphertext may be deleted after `expiry`

## Idempotency

- `PUT /v1/readers/{reader_id}` is idempotent for the same `(controller,
  reader_pubkey)`
- `POST /v1/disclosures` is not idempotent by payload alone
- replay protection is enforced by `(controller, nonce)`

## `SystemCiphertextV1`

The coordinator treats the coprocessor output as an opaque type defined by the
`MPC` specification.

```rust
#[derive(Clone, Debug, PartialEq, Eq)]
pub struct SystemCiphertextV1 {
    pub key_id: Bytes32,
    pub enc: Vec<u8>,
    pub wrapped_key: Vec<u8>,
    pub nonce: [u8; 12],
    pub ciphertext: Vec<u8>,
    pub aad: Vec<u8>,
}
```
