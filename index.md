# Symbolic Execution Specifications

`Symbolic Execution` is an open specification effort for private, composable
execution on Ethereum.

The project is inspired by the pointer-based confidential token model described
in [ERC-7984](https://eips.ethereum.org/EIPS/eip-7984):
contracts manipulate private values as opaque objects, those values compose
across applications, and the realization of private computation happens
asynchronously off-chain.

This repository starts from the core abstraction, not from a fully decomposed
infrastructure map.

The first objective is to describe the system clearly enough that contributors
can reason about it, challenge it, and eventually implement it.

The broader aim is to define a credibly neutral privacy layer for Ethereum:
shared privacy infrastructure that remains open, composable, and legible enough
to be adopted across applications rather than controlled by a single silo.

## Core Thesis

Private computation on Ethereum should feel like a native contract primitive,
not like an external oracle pattern.

That means:

- private values should appear on-chain as opaque references rather than
  plaintext
- contracts should be able to express operations over those values
- private state should remain composable across applications
- actual private computation can happen off-chain as long as the contract-facing
  abstraction remains stable and verifiable.

## Current Direction

The current architecture direction is a hybrid model:

- attested execution for private computation (TEEs)
- threshold cryptography for key custody, decryption, re-encryption, and
  resilience (MPC)

This is not presented as a finalized implementation commitment yet. It is the
current direction of the specification effort.

## System Pieces

The current minimal architecture has four pieces:

- `Client`: the user-facing or application-facing component that originates
  private inputs and receives authorized outputs
- `symVM`: the contract-facing model that exposes handles, symbolic operations,
  and disclosure flows on-chain
- `Coprocessor`: the off-chain private execution layer that resolves symbolic
  work in an attested environment
- `MPC`: the threshold-cryptography layer for key custody, decryption,
  re-encryption, and resilience

Each piece has its own specification, but the first and most developed one is
still `symVM`.

## How The Pieces Connect

The current high-level flow is:

1. The `Client` prepares private inputs and submits transactions to contracts
   that use `symVM`.
2. `symVM` gives contracts a stable on-chain abstraction based on private
   handles and symbolic operations.
3. The `Coprocessor` observes those `symVM` requests, tracks the symbolic
   dependency graph, and resolves private execution off-chain.
4. When a flow requires key operations, the `Coprocessor` coordinates with
   `MPC`.
5. `MPC` performs threshold decryption, re-encryption, or signing work without
   making the coprocessor a unilateral key holder.
6. The resolved result either becomes materialized private state, is disclosed
   privately to a reader, or is returned to a contract through an explicit
   callback flow.

## `symVM`

`symVM` is the contract-facing model of the system.

It defines how contracts represent private values, express private operations,
and participate in later disclosure or finalization flows when needed.

The purpose of `symVM` is to describe the developer-facing interface separately
from the execution and cryptographic details behind it.

## Design Goals

- Preserve composability of private state across contracts.
- Keep private values represented as opaque on-chain objects rather than
  plaintext.
- Keep the developer-facing abstraction stable while allowing the backend to evolve as the private computation model matures.
- Start with a simple model that can become more trust-distributed over time.

## Scope Of This Spec

This spec starts as a public design document.

It should gradually become more precise by defining:

- the core terminology of the system
- the meaning of private values and symbolic operations
- how the client, `symVM`, coprocessor, and `MPC` connect
- the lifecycle of a private operation from expression to materialization, and
  to disclosure when requested
- the permissions model for reads and disclosure
- the first reusable application patterns built on top

## Lower-Level Specs

- [`./client/README.md`](./client/README.md)
- [`./coprocessor/README.md`](./coprocessor/README.md)
- [`./mpc/README.md`](./mpc/README.md)
- [`./symvm/README.md`](./symvm/README.md)
- [`./symvm/symvm-private-handles.md`](./symvm/symvm-private-handles.md)
- [`./symvm/symvm-operations.md`](./symvm/symvm-operations.md)
- [`./symvm/symvm-operation-lifecycle.md`](./symvm/symvm-operation-lifecycle.md)
- [`./symvm/symvm-permissions-and-reads.md`](./symvm/symvm-permissions-and-reads.md)

## Non-Goals For The First Draft

- fully specifying production infrastructure
- proving every security property formally
- locking in all backend details before the contract model is clear
- standardizing multiple application interfaces too early
