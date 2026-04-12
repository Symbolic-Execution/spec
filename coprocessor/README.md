# Coprocessor Specification

## Goal

Define the minimal role of the coprocessor in the `Symbolic Execution` stack.

This draft keeps the coprocessor spec intentionally small. For now, it only
states what the coprocessor is responsible for relative to `symVM` and `MPC`.

## Working Model

The coprocessor is the off-chain private execution layer.

It observes `symVM` requests, tracks the symbolic dependency graph, resolves
private computation in an attested environment, and prepares outcomes for
materialization or disclosure.

The current direction is to run the execution environment inside a `TEE`.

## Minimum Responsibilities

- observe the symbolic requests emitted or implied by `symVM`
- track issued handles, pending symbolic state, and disclosure requests
- persist a local copy of the private handle registry and execution metadata
- resolve symbolic computations in the private execution environment
- produce attested results for later verification or disclosure
- coordinate with `MPC` when key operations are required

## Persistence Model

For now, the minimal working assumption is:

- each coprocessor instance maintains its own local database
- there is no required shared mutable database across coprocessors
- decentralization comes from shared handle semantics and recoverable inputs,
  not from a single shared storage system

This keeps the architecture simple while leaving room for later replication and
data-availability work.

## Interfaces

### Coprocessor <- `symVM`

The coprocessor consumes:

- handle identifiers
- symbolic operation requests
- disclosure requests
- any on-chain metadata needed to reconstruct the symbolic graph

### Coprocessor <-> `MPC`

The coprocessor uses the `MPC` API when a flow requires:

- decryption
- re-encryption to a reader
- threshold signatures or equivalent authorization artifacts

The base spec does not require the coprocessor to expose a direct public
interface to `sym-client`.
