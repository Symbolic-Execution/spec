# `MPC` Specification

## Role

`MPC` is the threshold key custody and ciphertext transformation service. It
exposes an internal HTTP API used by the `Coordinator` and the coprocessor.

Enclave authorization is based on attestation that binds the enclave public
key to an approved enclave measurement.

## Responsibilities

- hold key shares rather than a unilateral decryption key
- publish the public configuration used for encryption and ciphertext
  transformation
- register reader public keys
- transform `SystemCiphertextV1` to `EnclaveCiphertextV1` for an approved
  enclave key
- transform `SystemCiphertextV1` to `ReaderCiphertextV1` for a registered
  reader key
- define the ciphertext envelopes used by the rest of the system

## Interfaces

### `Coordinator` -> `MPC`

The coordinator uses `MPC` to:

- fetch the public configuration bundle
- register or rotate reader keys
- transform `SystemCiphertextV1` to `ReaderCiphertextV1`

### Coprocessor -> `MPC`

The coprocessor uses `MPC` to:

- transform `SystemCiphertextV1` to `EnclaveCiphertextV1` for an attested
  enclave key

## Detailed Specs

- [`./mpc-api.md`](./mpc-api.md)

## Trust Boundaries

- key shares remain inside `MPC`
- `MPC` returns ciphertexts, not raw decryption material
- `MPC` does not perform symbolic execution
- enclave authorization is based on an approved measurement and a valid
  attestation for the enclave public key
