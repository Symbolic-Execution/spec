# `symVM` Operations

## Goal

Define the minimum symbolic operation surface of `symVM`.

This document focuses on the smallest set of operations needed to express a
confidential token model using the initial typed handles defined in
[`symvm-private-handles.md`](./symvm-private-handles.md).

## Working Model

A `symVM` operation consumes one or more symbolic handles and returns a new
symbolic handle representing the result of that operation.

At the contract interface level:

- operations are typed
- operations do not reveal plaintext
- operations return fresh symbolic results
- operations do not synchronously resolve private predicates into ordinary EVM
  booleans

In the current working model, contracts express symbolic intent immediately,
while concrete private execution and validation happen later in the system.

## Initial Handle Types

This document assumes the initial handle types are:

- `suint256`
- `sbool`

No broader type surface is required yet.

## Requirements

### Typed

Each operation must define its valid input and output handle types.

### Opaque

No operation should expose the plaintext value of its inputs or outputs to the
contract.

### Composable

The output of one operation must be usable as the input to subsequent
operations.

### Interface-Level Purity

At the contract interface level, symbolic operations should behave like pure
transformations over handles. They should not require the contract to reason
about backend execution details.

### No Synchronous Private Branching

Operations returning `sbool` must not be treated as ordinary Solidity `bool`
values. Private predicates remain symbolic until a later disclosure,
authorization, or finalization step.

## Initial Operation Surface

The first operation surface should be limited to what is needed for confidential
token balances, transfer amounts, and token-related predicates.

### Arithmetic Over `suint256`

- `add(suint256, suint256) -> suint256`
- `sub(suint256, suint256) -> suint256`

These operations are sufficient to express the balance transitions required by
mint, transfer, and allowance-style flows.

### Comparisons Over `suint256`

- `eq(suint256, suint256) -> sbool`
- `lt(suint256, suint256) -> sbool`
- `lte(suint256, suint256) -> sbool`
- `gt(suint256, suint256) -> sbool`
- `gte(suint256, suint256) -> sbool`

These operations are sufficient to express core predicates such as balance
checks, allowance checks, and equality conditions over private values.

### Boolean Operations Over `sbool`

- `and(sbool, sbool) -> sbool`
- `or(sbool, sbool) -> sbool`
- `not(sbool) -> sbool`

These operations are sufficient to combine private predicates without requiring
plaintext disclosure at the point where the contract expresses intent.

## Semantics Of Operation Results

Every symbolic operation returns a fresh handle representing a derived private
value or derived private predicate.

In the current working model:

- input handles remain valid after the operation is expressed
- output handles may be stored, emitted, or passed to other contracts
- an `sbool` result is still a handle and not an immediately inspectable branch
  condition
- acceptance or rejection of operations that depend on private validity belongs
  to a later stage of the system, not to ordinary Solidity control flow

## Not Included Yet

The first operation surface does not yet define:

- multiplication or division
- mixed public and symbolic arithmetic
- symbolic selection such as `select(pred, a, b)`
- aggregate operations
- bitwise operations
- explicit literal constructors

These can be added later if they prove necessary for the core model or for
higher-level standards.

## Minimal Token-Oriented Surface

The current working choice is to keep the initial operation surface narrowly
optimized for a confidential token model.

This avoids importing a large symbolic standard library before the semantics of
request, validation, and finalization are clear.
