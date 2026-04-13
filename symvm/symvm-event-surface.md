# `symVM` Event Surface

## Scope

Define the canonical on-chain event surface emitted by `symVM` for coprocessor
ingestion.

This event surface is the integration boundary between `symVM` and the
coprocessor host. It defines the minimum information required to reconstruct
handle lineage, private input sources, and symbolic operation dependencies.

## Event Principles

The event surface satisfies the following requirements:

- the coprocessor can reconstruct handle lineage from `symVM` logs alone
- the coprocessor does not need to decode arbitrary application calldata
- event order is the canonical order of symbolic expression
- every derived handle has exactly one originating `symVM` event
- input handle events carry the encrypted payload needed to seed off-chain
  execution
- operation semantics follow [`./symvm-operations.md`](./symvm-operations.md)

## Core Types

```rust
pub type Address = [u8; 20];
pub type Bytes32 = [u8; 32];

pub struct DomainId(pub Bytes32);
pub struct HandleId(pub Bytes32);

pub enum HandleType {
    Suint256 = 1,
    Sbool = 2,
}

pub enum OperationCode {
    Add = 1,
    Sub = 2,
    Eq = 3,
    Lt = 4,
    Lte = 5,
    Gt = 6,
    Gte = 7,
    And = 8,
    Or = 9,
    Not = 10,
    Select = 11,
}
```

`HandleType` maps to the canonical `type_tag` strings used in ciphertext
bindings:

- `HandleType::Suint256` -> `"suint256"`
- `HandleType::Sbool` -> `"sbool"`

## Event Order

The canonical ingestion order is log order within the canonical chain view.

For the coprocessor:

- consume only logs from the `safe` chain view by default
- identify each log by `(chain_id, block_hash, tx_hash, log_index)`
- process logs in ascending `(block_number, tx_index, log_index)` order
- discard derived state from orphaned blocks

## Event Types

### `HandleImportedV1`

`HandleImportedV1` creates a handle from a client-provided private input.

```rust
pub struct HandleImportedV1 {
    pub domain_id: DomainId,
    pub contract: Address,
    pub handle_id: HandleId,
    pub handle_type: HandleType,
    pub system_ciphertext: Vec<u8>,
}
```

Field semantics:

- `domain_id` identifies the `symVM` domain
- `contract` is the contract that invoked `symVM`
- `handle_id` is the newly created handle
- `handle_type` is the contract-level handle type
- `system_ciphertext` is the canonical-CBOR encoding of `SystemCiphertextV1`

Ingestion rules:

- `handle_id` becomes a ready source handle
- `system_ciphertext` seeds the off-chain value for that handle
- the coprocessor records the handle type from `handle_type`

### `OperationRequestedV1`

`OperationRequestedV1` creates a derived handle from one symbolic operation.

```rust
pub struct OperationRequestedV1 {
    pub domain_id: DomainId,
    pub contract: Address,
    pub output_handle_id: HandleId,
    pub output_type: HandleType,
    pub operation: OperationCode,
    pub input_handles: Vec<HandleId>,
}
```

Field semantics:

- `domain_id` identifies the `symVM` domain
- `contract` is the contract that invoked `symVM`
- `output_handle_id` is the newly created derived handle
- `output_type` is the type of the derived handle
- `operation` identifies the symbolic operation
- `input_handles` is the ordered input list for the operation

Input ordering is part of the event semantics.

For `Select`, `input_handles` is ordered as:

- predicate
- when_true
- when_false

Ingestion rules:

- `output_handle_id` becomes a pending derived handle
- the coprocessor records `operation`, `input_handles`, and `output_type`
- the handle becomes executable once every input handle is ready

## Input Validation

`symVM` validates client-submitted ciphertexts on-chain at import time before
creating a handle and emitting `HandleImportedV1`.

Validation rules:

1. parse the `aad` field of the submitted `SystemCiphertextV1` as
   `SystemInputAadV1`
2. verify `system_ciphertext.key_id == aad.key_id`
3. verify `aad.contract == msg.sender`
4. verify `aad.key_id` matches the currently active system key ID
5. verify `aad.type_tag` matches the declared handle type
6. verify `aad.domain_id` matches the `symVM` domain
7. verify `aad.chain_id` matches the current chain

If any check fails, the import is rejected and no handle is created.

These checks provide early rejection of malformed or misbound ciphertexts.
The MPC will independently reject invalid ciphertexts at re-encryption time,
so the on-chain checks are not the sole line of defense — they prevent
wasting coprocessor work on inputs that will inevitably fail.

Re-submitting the same valid ciphertext in a separate transaction creates a
new handle with a new handle ID. The underlying private value is the same,
but the handle is distinct. This is not an attack — it is equivalent to
assigning the same value to two variables.

## Validity Rules

- `domain_id` must match the `symVM` domain that emitted the event
- a handle id must not be created more than once
- every input handle in `OperationRequestedV1` must refer to a previously
  observed handle in the same `domain_id`
- operation arity and input/output type rules must match
  [`./symvm-operations.md`](./symvm-operations.md)

## Event Emission Rules

- `symVM` emits exactly one `HandleImportedV1` for each imported encrypted
  input handle
- `symVM` emits exactly one `OperationRequestedV1` for each symbolic operation
  expression
- the emitted event is the canonical source for coprocessor ingestion

## Coprocessor Reconstruction

The coprocessor reconstructs the symbolic graph as follows:

1. consume `HandleImportedV1` and register ready source handles
2. consume `OperationRequestedV1` and register pending derived handles
3. validate operation arity and input/output types
4. mark a derived handle executable once all input handles are ready
5. execute the symbolic operation and bind the result to `output_handle_id`

## Not Included

The base event surface does not define:

- on-chain result completion events
- on-chain disclosure request events
- batch operation events
- alternate data-availability packaging for imported ciphertexts
