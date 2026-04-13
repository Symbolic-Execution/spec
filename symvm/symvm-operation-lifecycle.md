# `symVM` Operation Lifecycle

Every symbolic operation passes through three stages:

1. expression
2. off-chain resolution
3. completion

## Stage 1: Expression

During contract execution:

- the contract calls `symVM` over existing handles
- the contract receives a fresh handle immediately
- plaintext is not revealed
- private predicates are not turned into Solidity branch conditions
- symbolic intent is recorded on-chain for later resolution

## Stage 2: Off-Chain Resolution

The off-chain system observes the `symVM` event surface and reconstructs the
symbolic graph.

Resolution includes:

- loading the required private inputs
- evaluating the symbolic operations
- producing the result as `SystemCiphertextV1`
- producing any artifacts needed for later completion

## Stage 3: Completion

The base model has two completion modes:

- handle materialization
- reader-targeted disclosure

### Handle Materialization

- the resolved `SystemCiphertextV1` is bound to the derived handle
- no plaintext is revealed
- no on-chain completion transaction is required

### Reader-Targeted Disclosure

- the `Coordinator` verifies authorization
- the coprocessor returns the resolved `SystemCiphertextV1`
- `MPC` transforms it to `ReaderCiphertextV1`
- the authorized reader decrypts locally

No plaintext is posted on-chain just to complete the request.

Contract-visible completion is defined separately by higher-level standards.
