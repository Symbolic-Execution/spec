# `symVM` Operation Lifecycle

## Goal

Define how a symbolic operation moves from contract expression to private
resolution and completion.

This document exists to make the asynchronous model explicit. The private handle
and operation documents define what contracts can express. This document defines
how those expressions become resolved outcomes in the system.

## Working Model

`symVM` separates contract-time expression from backend-time realization.

At the contract interface level, a contract can create and compose symbolic
handles immediately. The concrete private computation behind those handles is
resolved later by the `Symbolic Execution` stack.

In the current working model, a symbolic operation passes through five stages:

1. expression
2. pending symbolic state
3. off-chain resolution
4. validation and policy checks
5. completion

## Stage 1: Expression

During contract execution, a contract invokes symbolic operations over existing
handles and receives fresh derived handles immediately.

At this stage:

- the contract does not receive plaintext results
- private predicates are not turned into ordinary Solidity branch conditions
- symbolic intent is recorded in a form that the rest of the system can resolve

The purpose of this stage is to preserve a synchronous contract-facing
programming model even though private realization is asynchronous.

## Stage 2: Pending Symbolic State

After expression, newly created handles exist as valid symbolic references even
if their concrete values have not yet been resolved.

This means:

- a derived handle may refer to a pending result
- contracts may further compose over pending handles
- the system may accumulate a dependency graph of symbolic computations

**NOTE:** This is a core property of the model. Without it, private computation would
fall back toward a request-per-step external service workflow rather than
remaining part of the contract abstraction.

## Stage 3: Off-Chain Resolution

The off-chain private computation system observes or receives the symbolic work that has been
requested and performs the corresponding computations.

This stage includes:

- resolving the relevant private inputs
- evaluating the requested symbolic operations
- producing concrete private outputs
- producing any execution artifacts needed for the next stage

This document does not yet require any specific execution backend. It only
requires that the symbolic requests become concretely evaluable by the system.

The intended behavior is:

- the executor resolves the pending symbolic work off-chain
- the result is bound to the already-issued symbolic handle

For `symVM`, the backend artifact should be a TEE-backed execution result plus
any threshold-crypto material needed for later disclosure.

## Stage 4: Validation And Policy Checks

Before a result is completed, the system may need to validate
additional conditions.

- track request identifiers and request classes
- replicate or observe the permission state needed for disclosure
- verify a valid EIP-712 signed disclosure request from the handle's
  controlling account when completion is reader-targeted
- verify the TEE execution receipt or attestation for the request
- verify threshold signatures when the request involves ciphertext
  transformation or re-encryption
- package the validated outcome for either handle materialization or off-chain
  delivery

## Stage 5: Completion

Completion is the step where resolved symbolic work becomes usable in one of the
completion modes supported by `symVM`.

Completion has two distinct modes:

- handle materialization without plaintext disclosure
- reader-targeted disclosure to a specific authorized reader

### Mode A: Handle Materialization

For ordinary symbolic computation, the executor resolves the computation
off-chain and binds the result to the already-issued derived handle.

In this mode:

- no on-chain completion is required
- no plaintext is revealed
- the result becomes available for further symbolic use

### Mode B: Reader Disclosure

When an authorized reader requests disclosure through the `Coordinator` for
off-chain consumption,
completion may terminate off-chain after validation.

In this mode:

- the result is delivered privately to the authorized reader
- no on-chain completion is required
- no plaintext is posted on-chain just to complete the request

If a higher-level ERC later needs a contract-visible completion path, it should
specify that separately rather than making it part of the base `symVM`
lifecycle.
