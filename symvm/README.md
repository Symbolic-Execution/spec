# `SymVM` Specifications

This directory contains more concrete specifications for `symVM`.

The goal of these documents is to define the contract-facing semantics of the
system in smaller pieces before expanding into broader implementation details.

## Current Sequence

Current:

1. [`symvm-private-handles.md`](./symvm-private-handles.md)
2. [`symvm-operations.md`](./symvm-operations.md)
3. [`symvm-operation-lifecycle.md`](./symvm-operation-lifecycle.md)
4. [`symvm-permissions-and-reads.md`](./symvm-permissions-and-reads.md)

## Working Style

These documents use a simple convention:

- `Goal` explains why the document exists
- `Working model` describes the current proposed semantics
- `Requirements` list the properties the design must preserve
- `Open questions` identify the next decisions that still need to be made
