# `MPC` Specification

## Goal

Define the minimal role of `MPC` in the `Symbolic Execution` stack.

This draft keeps `MPC` narrowly scoped to key custody and threshold
authorization. It does not treat `MPC` as the general private execution layer.

## Working Model

`MPC` is the threshold-cryptography layer responsible for key custody,
decryption authorization, re-encryption, and resilience.

In the current architecture direction:

- the coprocessor performs private execution
- `MPC` controls sensitive key operations

This separates computation from unilateral key power.

## Minimum Responsibilities

- hold key shares rather than a single decryption key
- participate in authorized decryption or re-encryption flows
- produce threshold artifacts needed for disclosure or verification
- avoid giving any single operator unilateral control over private outputs

## Interfaces

### `MPC` <- Coprocessor

`MPC` receives requests for key operations after the coprocessor has produced
the required execution context and attestation material.

### `MPC` -> Client

For private reader disclosure, `MPC` may help re-encrypt or release an output
to the authorized reader key.

### `MPC` -> Contract Flow

For contract-visible disclosure, `MPC` may contribute threshold artifacts that
are included in the callback or finalizer package submitted on-chain.

## Non-Goals For This Draft

- specifying a concrete `MPC` protocol
- specifying committee formation or slashing
- specifying the exact message formats exchanged with the coprocessor
