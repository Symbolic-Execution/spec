# Symbolic Execution

The open stack for private, composable Ethereum execution.

`Symbolic Execution` is an open-source effort exploring how privacy can become
part of the native application model on Ethereum, not just an external service, rollup
or isolated tool creating fragmented liquidity.

We are interested in systems where contracts can work with private values as
first-class objects and private state can compose across applications.

We aim to help define a credibly neutral privacy layer for Ethereum: open,
composable infrastructure that many applications can share rather than a
privacy system tied to a single product, operator, or silo.

## Why

Privacy on Ethereum is still too fragmented.

Many approaches are powerful in isolation, but hard to compose into a shared
application model for tokens, DeFi, coordination, and general-purpose contract
systems.

We think private computation should be easier to reason about, easier to build
with, and easier to plug into existing sources of liquidity, like Ethereum mainnet.

## Approach

We are focused on open specifications, public design work, and
reference implementations.

The current direction is inspired in part by
[ERC-7984: Confidential Fungible Token](https://eips.ethereum.org/EIPS/eip-7984),
especially its pointer-based and technology-agnostic approach to confidential
values, while aiming to define an independent and openly developed model for
private, composable execution on Ethereum.

## Spec Structure

The specifications are currently split into four minimal architecture modules:

- `client/` for the user-facing and application-facing role
- `symvm/` for the contract-facing abstraction
- `coprocessor/` for off-chain private execution
- `mpc/` for threshold key custody and authorization

## Open Source Ethos

We want to build this in public.

That means:

- open specifications
- public design discussions
- transparent tradeoffs
- iterative prototypes
- reusable components when possible

The goal is not just to publish code. It is to make the ideas, interfaces, and
design decisions legible enough that other people can challenge them, implement
them, and build on top of them.
