# `symVM` Permissions And Reads

Possession of a handle is not permission to read its plaintext.

## Base Read Model

- read access is authorized explicitly
- disclosure targets an authorized reader rather than publishing plaintext
  on-chain by default
- read requests are asynchronous
- private reader requests are submitted off-chain to the `Coordinator`, not to
  `symVM`

The base model defines only off-chain reader-targeted disclosure.

## Authorization

- disclosure uses an EIP-712 signed request from the handle's controlling
  account, as defined by the higher-level standard
- the coordinator checks current on-chain policy

The base model has no persistent approval registry.

## Fulfillment

- the `Coordinator` verifies the signed request
- the backend resolves the handle if needed
- `MPC` returns `ReaderCiphertextV1`
- the authorized reader decrypts locally

Contract-visible disclosure is defined separately by higher-level standards.
