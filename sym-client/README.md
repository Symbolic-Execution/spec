# `sym-client` SDK Specification

## Role

`sym-client` is the user-facing SDK. It prepares private inputs, submits
transactions to contracts that use `symVM`, manages reader keys, and decrypts
authorized private outputs returned through the `Coordinator`.

`sym-client` uses the Ethereum signing key for authorization and a separate
reader keypair for decryption.

## Responsibilities

- fetch the public configuration bundle from the `Coordinator`
- encode private plaintext values and encrypt them as `SystemCiphertextV1`
- submit transactions to contracts that use `symVM`
- track handle ids, reader ids, and disclosure request ids
- generate, register, and rotate reader keypairs
- sign reader registration and disclosure requests with the user's Ethereum
  signing key
- poll disclosure status when a request remains pending
- decrypt `ReaderCiphertextV1` locally with the reader secret key

## Interfaces

### `sym-client` -> `symVM`

`sym-client` submits transactions to contracts that use `symVM`.

Transaction usage includes:

- create new private handles
- pass existing handles
- express symbolic operations
- rely on off-chain disclosure requests for the base read flow

### `sym-client` -> `Coordinator`

`sym-client` uses the `Coordinator` API to:

- fetch the public configuration bundle
- register or rotate reader keys
- submit signed disclosure requests
- poll pending disclosure requests
- receive `ReaderCiphertextV1`

## Detailed Specs

- [`./sym-client-api.md`](./sym-client-api.md)

## Trust Boundaries

- the Ethereum signing key authenticates client requests
- the reader secret key decrypts reader-targeted ciphertexts
- `sym-client` does not perform private execution
- `sym-client` does not receive raw key material from `MPC`
