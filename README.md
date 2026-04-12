# Symbolic Execution Specifications

`Symbolic Execution` specifies a model for private, composable execution on
Ethereum.

The current design is informed by
[ERC-7984](https://eips.ethereum.org/EIPS/eip-7984): private values appear
on-chain as opaque references, compose across contracts, and are realized
asynchronously off-chain. This repository documents that model, starting from
the contract-facing abstraction and the minimum set of supporting system roles.

## Core Model

Private computation on Ethereum should look like a contract primitive rather
than an external oracle pattern.

That means:

- private values appear on-chain as opaque handles rather than plaintext
- contracts can express operations over those values
- private state can compose across applications
- private execution can happen off-chain while the contract-facing abstraction
  remains stable

## Architecture Modules

The current minimal architecture has four pieces:

- `sym-client`: SDK that prepares private inputs, manages reader keys, and
  receives authorized outputs
- `symVM`: exposes private handles, symbolic operations, and disclosure flows
  on-chain
- `Coprocessor`: resolves symbolic work off-chain
- `MPC`: provides threshold key custody, decryption, re-encryption, and
  related authorization flows

## High-Level Flow

1. The `sym-client` SDK prepares private inputs and submits transactions to contracts
   that use `symVM`.
2. `symVM` gives contracts the on-chain abstraction for private handles and
   symbolic operations.
3. The `Coprocessor` observes those requests, tracks dependencies, and resolves
   private execution off-chain.
4. When a flow requires key operations, the `Coprocessor` and `sym-client`
   call `MPC` over HTTP.
5. `MPC` performs threshold decryption, re-encryption, key registration, or
   signing work.
6. The resolved result is materialized as private state or disclosed to an
   authorized reader.

## Scope

This specification currently focuses on:

- core terminology
- private values and symbolic operations
- how `sym-client`, `symVM`, `Coprocessor`, and `MPC` connect
- operation lifecycle from expression to materialization or disclosure
- permissions for reads and disclosure
- reusable application patterns built on top of the model

## Specs

- [`./sym-client/README.md`](./sym-client/README.md)
- [`./coprocessor/README.md`](./coprocessor/README.md)
- [`./mpc/README.md`](./mpc/README.md)
- [`./symvm/README.md`](./symvm/README.md)
- [`./symvm/symvm-private-handles.md`](./symvm/symvm-private-handles.md)
- [`./symvm/symvm-operations.md`](./symvm/symvm-operations.md)
- [`./symvm/symvm-operation-lifecycle.md`](./symvm/symvm-operation-lifecycle.md)
- [`./symvm/symvm-permissions-and-reads.md`](./symvm/symvm-permissions-and-reads.md)
