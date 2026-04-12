# `symVM` Permissions And Reads

## Scope

Define how `symVM` separates handle possession from authorization to read the
underlying private value.

## Read Model

In `symVM`, possession of a handle is not permission to read its plaintext.

Reads use the following model:

- read access is authorized explicitly
- disclosure targets an authorized reader rather than publishing plaintext
  on-chain by default
- read requests are asynchronous
- private reader requests are submitted off-chain to the `Coordinator`, not to
  `symVM`

The base model defines only off-chain reader-targeted disclosure.

## Authorization

Disclosure authorization is based on:

- an EIP-712 signed request from the handle's controlling account, as defined
  by the higher-level standard
- coordinator checks against current on-chain policy

The base model has no persistent approval registry.

## Requirements

The permission model must remain:

- explicit: handle possession alone is not enough
- reader-targeted: the default disclosure target is a specific authorized
  reader
- separable from handle shape: permissions can evolve without changing handle
  types
- composable: higher-level standards can define richer delegation patterns
- non-public by default: plaintext is not revealed as a side effect of an
  ordinary read

## Fulfillment

For reader-targeted disclosure:

- the `Coordinator` verifies the signed request
- the backend resolves the handle if needed
- `MPC` returns `ReaderCiphertextV1`
- the authorized reader decrypts locally

A higher-level ERC that needs contract-visible disclosure defines that
separately.
