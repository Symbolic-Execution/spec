# `symVM` Private Handles

## Model

A private handle is an opaque on-chain reference to a private value in a
`symVM` domain.

A handle:

- can be stored, passed, and used as input to symbolic operations
- does not reveal plaintext
- is not a ciphertext object
- does not by itself authorize disclosure
- remains valid while the underlying private value is live

The initial handle types are:

- `suint256`
- `sbool`

## Solidity Representation

Handle types are Solidity user-defined value types wrapping `bytes32`:

```solidity
type suint256 is bytes32;
type sbool is bytes32;
```

This keeps handles opaque and type-safe at zero runtime gas cost.
The `SYM` library may use `.wrap()` and `.unwrap()` internally when it needs
raw `bytes32` values.

`bytes32(0)` is an uninitialized handle. `symVM` rejects uninitialized handles
as operation inputs.

## Handle ID Derivation

`symVM` assigns handle IDs. Contracts do not choose them.

```rust
pub fn derive_handle_id(
    domain_id: DomainId,
    contract: Address,
    handle_nonce: u64,
) -> HandleId {
    HandleId(keccak256(abi.encode(domain_id, contract, handle_nonce)))
}
```

The preimage is `abi.encode(domain_id, contract, handle_nonce)` with
`domain_id` as `bytes32`, `contract` as `address`, and `handle_nonce` as
`uint64`.

`handle_nonce` is a per-`(domain_id, contract)` counter. It starts at `0`.
`symVM` reads the current counter, derives the handle ID, then increments the
counter by one.

This makes handle IDs:

- deterministic from on-chain state
- unique for a given contract in a given domain
- verifiable by the coprocessor from emitted events

## What Contracts Can Assume

- a contract may store a handle in state
- a contract may pass a handle to another contract
- a contract may use an authorized handle in later symbolic operations
- a contract receives fresh handles from imports, plaintext conversion, and symbolic operations
- disclosure authorization is defined separately in
  [`./symvm-permissions-and-reads.md`](./symvm-permissions-and-reads.md)
