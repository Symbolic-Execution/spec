# `Coordinator` Specification

## Goal

Define the role of the `Coordinator` in the `Symbolic Execution` stack.

The `Coordinator` is the off-chain control plane that sits between
`sym-client` and the backend services. It is responsible for authorization,
request lifecycle management, and routing between the coprocessor and `MPC`.

## Working Model

The `Coordinator` is the public HTTP API used by `sym-client` for asynchronous
operations.

It is not a key holder and it is not the private execution environment. Its
job is to:

- accept signed user requests
- verify authorization against current on-chain state and protocol metadata
- create request identifiers and track async status
- dispatch unresolved-handle work to the coprocessor
- dispatch key operations to `MPC`
- return status or reader-targeted ciphertext packages to `sym-client`

Disclosure authorization uses the following model:

- a disclosure request is authorized by a valid EIP-712 signature from the
  handle's controlling account, as defined by the higher-level standard
- no persistent off-chain approval registry is required

## Minimum Responsibilities

- expose the public API consumed by `sym-client`
- publish the current system public configuration needed for input encryption
- verify user signatures, nonces, expiries, and reader-key bindings
- inspect on-chain controller state and any higher-level-standard disclosure
  policy
- create and track async disclosure request state
- ask the coprocessor to resolve a handle when the result is not yet ready
- ask `MPC` to transform ciphertexts to either a reader key or another
  authorized recipient key such as an attested enclave key
- return `ReaderCiphertextV1` immediately when ready, or request status when
  the request remains pending

## Interfaces

### `sym-client` -> `Coordinator`

The minimal public interface is:

- `GET /v1/config`
- `PUT /v1/readers/{reader_id}`
- `POST /v1/disclosures`
- `GET /v1/disclosures/{request_id}` for pending requests

`POST /v1/disclosures` returns:

- return `200` with `ReaderCiphertextV1` when the disclosure is already ready
- return `202` with a `request_id` when the disclosure remains pending

The signed disclosure request uses EIP-712 with the user's Ethereum signature
and binds at least:

- `chain_id`
- `contract`
- `handle_id`
- `reader_id`
- `nonce`
- `expiry`

### `Coordinator` -> `symVM` / Chain

The coordinator reads:

- current contract state needed to identify the handle's controlling account
  and any higher-level-standard disclosure policy
- logs or metadata needed to resolve handle lineage

### `Coordinator` -> Coprocessor

The coordinator asks the coprocessor to:

- resolve a handle whose value is not yet materialized
- return the current `SystemCiphertextV1`
- return receipts, status, and failure information for async jobs

### `Coordinator` -> `MPC`

The coordinator asks `MPC` to:

- publish the current system public configuration
- register or rotate reader public keys if the deployment uses a persistent
  reader registry
- transform a system-held ciphertext into `ReaderCiphertextV1`
- produce threshold signatures or equivalent authorization artifacts when
  needed

## Trust Boundaries

The coordinator is policy and workflow infrastructure, not a cryptographic
trust anchor.

That means:

- it does not receive plaintext private values
- it does not hold unilateral decryption keys
- it may observe request metadata and access patterns
- it remains replaceable without changing the `symVM` contract abstraction
