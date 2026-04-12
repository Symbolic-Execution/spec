# `Coordinator` Specification

## Role

The `Coordinator` is the public HTTP API between `sym-client` and the backend
services. It is responsible for authorization, request lifecycle management,
and routing between the coprocessor and `MPC`.

The coordinator:

- accepts signed user requests
- verifies authorization against current on-chain state and protocol metadata
- creates request identifiers and tracks async status
- dispatches unresolved-handle work to the coprocessor
- dispatches key operations to `MPC`
- returns status or reader-targeted ciphertext packages to `sym-client`

Disclosure authorization is based on a valid EIP-712 signature from the
handle's controlling account, as defined by the higher-level standard.

## Responsibilities

- expose the public API consumed by `sym-client`
- publish the system public configuration needed for input encryption
- verify user signatures, nonces, expiries, and reader-key bindings
- inspect on-chain controller state and any higher-level-standard disclosure
  policy
- create and track async disclosure request state
- ask the coprocessor to resolve a handle when the result is not yet ready
- ask `MPC` to transform ciphertexts to a reader key
- return `ReaderCiphertextV1` immediately when ready, or request status when
  the request remains pending

## Interfaces

### `sym-client` -> `Coordinator`

Public endpoints:

- `GET /v1/config`
- `PUT /v1/readers/{reader_id}`
- `POST /v1/disclosures`
- `GET /v1/disclosures/{request_id}` for pending requests

`POST /v1/disclosures` returns:

- return `200` with `ReaderCiphertextV1` when the disclosure is already ready
- return `202` with a `request_id` when the disclosure remains pending

The signed disclosure request uses EIP-712 with the user's Ethereum signature
and binds:

- `chain_id`
- `contract`
- `handle_id`
- `reader_id`
- `nonce`
- `expiry`

### `Coordinator` -> `symVM` / Chain

The coordinator reads:

- contract state needed to identify the handle's controlling account and any
  higher-level-standard disclosure policy

### `Coordinator` -> Coprocessor

The coordinator asks the coprocessor to:

- resolve a handle whose value is not yet materialized
- return `SystemCiphertextV1`
- return receipts, status, and failure information for async jobs

### `Coordinator` -> `MPC`

The coordinator asks `MPC` to:

- publish the system public configuration
- register or rotate reader public keys
- transform a system-held ciphertext into `ReaderCiphertextV1`

## Detailed Specs

- [`./coordinator-api.md`](./coordinator-api.md)

## Trust Boundaries

The coordinator is policy and workflow infrastructure, not a cryptographic
trust anchor.

- it does not receive plaintext private values
- it does not hold unilateral decryption keys
- it observes request metadata and access patterns
- it remains replaceable without changing the `symVM` contract abstraction
