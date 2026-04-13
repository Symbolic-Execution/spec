# `symVM` Private Handles

## Scope

Define what a private handle is in `symVM` and what contracts can assume about
it.

## Handle Semantics

A private handle is an opaque on-chain reference to a private value managed by
the `Symbolic Execution` stack.

At the contract interface level, a handle:

- identifies a private value within a `symVM` domain
- can be stored, passed, and used as input to symbolic operations
- does not reveal the underlying plaintext
- does not by itself authorize plaintext disclosure
- remains meaningful independently of the backend used to realize the value

A handle is a reference, not a bearer capability and not a ciphertext object.

## Required Properties

A private handle must be:

- opaque: the handle value does not reveal the plaintext
- stable: the reference remains valid while the private value is live
- composable: the handle can move across contract boundaries
- typed: the contract interface knows the handle type
- non-authorizing: possession of the handle does not grant read access

## Initial Handle Types

The initial handle type surface is:

- `suint256`
- `sbool`

These types cover the minimum confidential token model.

## Solidity Representation

Handle types are Solidity user-defined value types wrapping `bytes32`:

```solidity
type suint256 is bytes32;
type sbool is bytes32;
```

This representation:

- makes handles opaque at the language level — arithmetic and comparison
  operators do not apply
- provides compile-time type safety between `suint256`, `sbool`, and raw
  `bytes32` at zero runtime gas cost
- allows handles to be stored in mappings, passed as function arguments, and
  emitted in events using standard Solidity patterns
- uses `.wrap()` and `.unwrap()` for conversions when the library needs to
  operate on raw `bytes32` internally

A zero-valued handle (`bytes32(0)`) represents an uninitialized handle.
`symVM` operations treat uninitialized handles as invalid inputs.

## Handle ID Derivation

`symVM` assigns handle IDs deterministically. The contract does not choose
the handle ID — it receives it as the return value of the operation.

```rust
pub fn derive_handle_id(
    domain_id: DomainId,
    contract: Address,
    handle_nonce: u64,
) -> HandleId {
    HandleId(keccak256(abi.encode(domain_id, contract, handle_nonce)))
}
```

The hash input is the Solidity `abi.encode` of the three fields in order:
`domain_id` as `bytes32`, `contract` as `address`, `handle_nonce` as
`uint64`. This is the standard ABI encoding with each value padded to 32
bytes.

`handle_nonce` is a per-contract counter maintained by `symVM`. It starts at
zero for each `(domain_id, contract)` pair. `symVM` reads the current counter
value, uses it to derive the handle ID, and then increments the counter by
one. The first handle created by a contract in a domain has `handle_nonce =
0`.

Derivation properties:

- uniqueness is guaranteed by the counter
- the handle ID is deterministic from on-chain state
- the coprocessor can verify handle IDs from event data
- contracts do not need to predict handle IDs in advance

## Contract Semantics

At the contract level:

1. a contract may store a handle in state
2. a contract may pass a handle to another contract
3. a contract may use one or more handles as inputs to symbolic operations
4. a contract may receive new handles as outputs from symbolic operations
5. a contract may participate in flows that later authorize disclosure for a
   handle

## Consequences

Because handles are references rather than capabilities:

- contracts can exchange handles without automatically delegating read access
- read authorization is defined elsewhere in the system
- logs and calldata containing handles are sensitive metadata, but not
  automatic declassification
