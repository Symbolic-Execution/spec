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
can reason about it, challenge it, and eventually implement it in public.

## Core Thesis

Private computation on Ethereum should feel like a native contract primitive,
not like an external oracle pattern.

That means:

- private values should appear on-chain as opaque references rather than
  plaintext
- contracts should be able to express operations over those values
- private state should remain composable across applications
- actual private computation can happen off-chain as long as the contract-facing
  abstraction remains stable

## Current Direction

The current architecture direction is a hybrid model:

- attested execution for private computation
- threshold cryptography for key custody, decryption, re-encryption, and
  resilience

This is not presented as a finalized implementation commitment yet. It is the
current direction of the specification effort.

## `symVM`

`symVM` is the contract-facing model of the system.

It defines how contracts represent private values, express private operations,
and receive finalized results.

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
- the lifecycle of a private operation from request to finalization
- the permissions model for reads and disclosure
- the first reusable application patterns built on top

## Non-Goals For The First Draft

- fully specifying production infrastructure
- proving every security property formally
- locking in all backend details before the contract model is clear
- standardizing multiple application interfaces too early
