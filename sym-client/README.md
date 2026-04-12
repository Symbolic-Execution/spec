# `sym-client` SDK Specification

## Goal

Define the minimal role of the `sym-client` SDK in the `Symbolic Execution`
stack.

This document stays intentionally small. Its purpose is to make the SDK's
responsibilities legible relative to `symVM`, the `Coordinator`, the
coprocessor, and `MPC`.

The package name may be `symbolic-execution-client`, but the specification
refers to the role as `sym-client`.

## Working Model

`sym-client` is the user-facing or application-facing SDK that prepares private
inputs, submits transactions to `symVM`, and receives authorized private
outputs.

`sym-client` is not the execution layer for symbolic programs. It prepares
private material, manages reader keys, and uses the `Coordinator` API for
asynchronous disclosure flows, while `symVM` remains the contract-facing
abstraction.

The base spec does not define a direct `sym-client` interface to the
coprocessor or to `MPC`.

## Minimum Responsibilities

- prepare private inputs and transaction parameters
- submit transactions that invoke `symVM` entrypoints
- track handles and request identifiers relevant to the user
- generate, register, or rotate reader keys
- sign asynchronous disclosure requests
- encode inputs and decode outputs using the specified ciphertext envelopes
- receive and decrypt authorized outputs returned through the `Coordinator`

## Interfaces

### `sym-client` -> `symVM`

The SDK calls contracts that use `symVM`.

Those calls may:

- create or pass private handles
- express symbolic operations
- submit disclosure-related transactions if a higher-level ERC requires them
- rely on off-chain disclosure requests for the base read flow

### `sym-client` -> `Coordinator`

The SDK speaks directly to the `Coordinator` API over HTTP.

The minimal interface is:

- fetch the current public-key and parameter bundle
- register or rotate reader keys
- submit signed disclosure requests
- poll asynchronous request status
- receive reader-targeted disclosure packages
