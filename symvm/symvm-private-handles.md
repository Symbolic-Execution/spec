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
