# Coprocessor Specification

## Goal

Define the minimal role of the coprocessor in the `Symbolic Execution` stack.

This document defines the coprocessor responsibilities relative to `symVM`,
the `Coordinator`, and `MPC`.

## Working Model

The coprocessor is the off-chain private execution layer.

It observes `symVM` requests, tracks the symbolic dependency graph, resolves
private computation in an attested environment, and prepares outcomes for
materialization or off-chain disclosure.

The execution environment runs inside a `TEE`.

## Minimum Responsibilities

- observe the symbolic requests emitted or implied by `symVM`
- track issued handles and pending symbolic state
- persist a local copy of the private handle registry and execution metadata
- resolve symbolic computations in the private execution environment
- produce attested results for later verification or disclosure
- expose result status and `SystemCiphertextV1` outputs to the `Coordinator`
- coordinate with `MPC` when key operations are required

## Persistence Model

- each coprocessor instance maintains its own local database
- there is no required shared mutable database across coprocessors
- decentralization comes from shared handle semantics and recoverable inputs,
  not from a single shared storage system

## Interfaces

### Coprocessor <- `symVM`

The coprocessor consumes:

- handle identifiers
- symbolic operation requests
- any on-chain metadata needed to reconstruct the symbolic graph

### `Coordinator` <-> Coprocessor

The coordinator may ask the coprocessor to:

- resolve a handle whose value is still pending
- return the current `SystemCiphertextV1`
- report job status, receipts, or failure information

### Coprocessor <-> `MPC`

The coprocessor uses the `MPC` API when a flow requires:

- authorized transformation of a system-held ciphertext to the enclave's
  attested public key
- threshold signatures or equivalent authorization artifacts
