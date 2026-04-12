# `symVM` Operations

## Scope

Define the symbolic operation surface exposed by `symVM`.

## Operation Model

A `symVM` operation consumes one or more handles and returns a fresh handle for
the result.

Operation properties:

- operations are typed
- operations do not reveal plaintext
- operations return fresh handles
- private predicates remain symbolic handles
- contracts express symbolic intent immediately, while resolution happens later

## Handle Types

The initial handle type surface is:

- `suint256`
- `sbool`

## Operations

### Arithmetic

- `add(suint256, suint256) -> suint256`
- `sub(suint256, suint256) -> suint256`

### Comparisons

- `eq(suint256, suint256) -> sbool`
- `lt(suint256, suint256) -> sbool`
- `lte(suint256, suint256) -> sbool`
- `gt(suint256, suint256) -> sbool`
- `gte(suint256, suint256) -> sbool`

### Boolean Operations

- `and(sbool, sbool) -> sbool`
- `or(sbool, sbool) -> sbool`
- `not(sbool) -> sbool`

### Selection

- `select(sbool, suint256, suint256) -> suint256`
- `select(sbool, sbool, sbool) -> sbool`

`select` is the private conditional-choice primitive of `symVM`.

It returns:

- the second input when the predicate is true
- the third input when the predicate is false

Selection rules:

- the first input is always `sbool`
- the second and third inputs have the same handle type
- the output type matches the second and third inputs
- the chosen branch is not observable to the contract at expression time

## Type Rules

Each operation defines:

- input arity
- valid input handle types
- output handle type

An operation is invalid if its inputs do not satisfy those rules.

## Result Semantics

Every operation returns a fresh derived handle.

At the contract interface level:

- input handles remain valid after the operation
- output handles may be stored, passed, or used in later operations
- `sbool` results are handles, not Solidity `bool` values
- `select` expresses private conditional choice without EVM control flow

## Not Included

This operation surface does not define:

- multiplication or division
- mixed public and symbolic arithmetic
- aggregate operations
- bitwise operations
- explicit literal constructors
