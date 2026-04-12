# `symVM` Operation Lifecycle

## Scope

Define how a symbolic operation moves from contract expression to off-chain
resolution and completion.

## Lifecycle Overview

`symVM` separates contract-time expression from backend-time realization.

Every symbolic operation passes through three stages:

1. expression
2. off-chain resolution
3. completion

## Stage 1: Expression

During contract execution, a contract invokes symbolic operations over existing
handles and receives fresh derived handles immediately.

At this stage:

- the contract does not receive plaintext results
- private predicates are not turned into ordinary Solidity branch conditions
- private conditional choice is expressed with symbolic operations such as
  `select`
- symbolic intent is recorded on-chain for later resolution

The result of expression is a new handle that may still be pending.

## Stage 2: Off-Chain Resolution

The off-chain private computation system observes the canonical `symVM` event
surface and reconstructs the symbolic dependency graph.

Resolution includes:

- loading the required private inputs
- evaluating the symbolic operations
- producing a concrete private result
- producing the execution artifacts needed for completion

For ordinary handle materialization, the result is packaged as
`SystemCiphertextV1` and bound to the already-issued handle.

## Stage 3: Completion

Completion makes the resolved result usable in one of the supported ways.

The base model has two completion modes:

- handle materialization
- reader-targeted disclosure

### Handle Materialization

For ordinary symbolic computation:

- the off-chain executor resolves the operation
- the result is bound to the derived handle
- no plaintext is revealed
- no on-chain completion transaction is required

### Reader-Targeted Disclosure

When an authorized reader requests disclosure through the `Coordinator`:

- the `Coordinator` verifies authorization
- the coprocessor returns the resolved `SystemCiphertextV1`
- `MPC` transforms the result to `ReaderCiphertextV1`
- the authorized reader decrypts locally

No plaintext is posted on-chain just to complete the request.

A higher-level ERC that needs a contract-visible completion path defines that
separately.
