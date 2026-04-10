# `symVM` Private Handles

## Goal

Define what a private handle is in `symVM` and what contracts can assume about
it.

This is the first lower-level specification because symbolic operations,
permissions, and finalization all depend on the handle model.

## Working Model

A private handle is an opaque on-chain reference to a private value managed by
the `Symbolic Execution` stack.

From the contract's perspective, a handle is a value that can be stored,
passed, and used as input to symbolic operations, but not directly inspected for
its underlying plaintext.

In the current working model:

- a handle identifies a private value within a given `symVM` domain
- a handle does not reveal the plaintext value it refers to
- a handle can be persisted in contract state and emitted in events
- a handle can be used as input to further symbolic operations
- possession of a handle alone does not imply permission to read its plaintext
- handles are typed at the contract interface level

## Requirements

A private handle should satisfy the following requirements:

### Opaque

Contracts and users must not be able to derive the private plaintext from the
handle value itself.

### Stable

A handle must remain a valid reference for as long as the underlying private
value is considered live by the system.

### Composable

Handles must be usable across contract boundaries so that private state can
participate in reusable application patterns rather than remaining trapped in a
single contract.

### Backend-Agnostic At The Interface

The meaning of a handle should not depend on whether the private value is
realized by one backend or another.

### Non-Authorizing By Default

The existence of a handle in calldata, memory, storage, or logs must not by
itself grant plaintext read access.

## What A Handle Represents

At the interface level, a handle represents a reference to a private value.

This document does not yet require the underlying implementation to use any
specific storage model, encryption format, or execution backend. It only
requires that the reference remain meaningful to the rest of `symVM`.

For now, the spec treats handles as references, not plaintext containers and
not ciphertext objects that contracts manipulate directly.

## Typed Handles

The current working choice is to use typed handles rather than a single generic
handle with separate type metadata.

This keeps contract interfaces easier to read and makes it simpler to define
which operations are valid for which values.

At the same time, the first version of the spec should avoid defining a large
type universe too early.

## Initial Type Surface

The initial type surface is limited to what is needed to specify a
confidential token model.

The current minimum set is:

- `suint256` for balances, transfer amounts, and allowances
- `sbool` for comparisons, predicates, and authorization-related symbolic
  results

Additional handle types can be added later if they become necessary for the
core model or for higher-level standards.

## Contract-Level Semantics

At the contract level, the minimum assumptions are:

1. A contract may store a handle in state.
2. A contract may pass a handle to another contract.
3. A contract may use one or more handles as inputs to symbolic operations.
4. A contract may receive new handles as outputs from symbolic operations.
5. A contract may participate in flows that eventually authorize disclosure or
   finalization related to a handle.

## Reference, Not Capability

The current working choice is to model a handle as a reference, not as a bearer
capability.

That means the handle points to private state, but authorization decisions are
made by explicit policy and permission logic elsewhere in the system.

This keeps the handle model simpler and makes it easier to reason about
composability. It also avoids treating leaked handle values as implicit
plaintext disclosure rights.

## Consequences Of This Choice

If handles are references rather than capabilities:

- contracts can exchange handles without automatically delegating read access
- read permissions need an explicit policy surface
- logs or calldata containing handles are sensitive metadata, but not automatic
  declassification
- the permission model can evolve without changing the basic handle type

## Naming Choice

The current naming direction is to prefer `symbolic`-style type names.

In practice, that means preferring names such as:

- `suint256`
- `sbool`

The reason is that these values are not plaintext private values inside the
contract. They are symbolic on-chain handles that refer to private values.

In prose, it is still fine to describe them as private handle types when
explaining the system at a high level.
