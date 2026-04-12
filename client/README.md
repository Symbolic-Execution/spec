# Client Specification

## Goal

Define the minimal role of the client in the `Symbolic Execution` stack.

This document stays intentionally small for now. Its purpose is only to make
the client's responsibilities legible relative to `symVM`, the coprocessor,
and `MPC`.

## Working Model

The client is the user-facing or application-facing component that originates
private inputs, submits transactions to `symVM`, and receives authorized
private outputs.

The client is not the execution layer for symbolic programs. It prepares
private material and participates in disclosure flows, while `symVM` remains
the contract-facing abstraction.

## Minimum Responsibilities

- prepare private inputs and transaction parameters
- submit transactions that invoke `symVM` entrypoints
- track the handles or request identifiers relevant to the user
- register or provide reader keys for private disclosure
- receive and decode authorized outputs returned to the user

## Interfaces

### Client -> `symVM`

The client calls contracts that use `symVM`.

Those calls may:

- create or pass private handles
- express symbolic operations
- request disclosure or callback-based finalization

### Client -> Coprocessor

The client does not need a permanently trusted direct control channel to the
coprocessor.

The minimal assumption is that the client can:

- observe status through chain events or protocol metadata
- receive private outputs once disclosure has been authorized

### Client -> `MPC`

The client may provide or register reader keys that `MPC` can target for
authorized disclosure.

The current spec does not yet require the client to speak directly to `MPC`
for ordinary symbolic execution.

## Non-Goals For This Draft

- specifying wallet UX
- specifying ciphertext formats
- specifying transport protocols between the client and backend services
