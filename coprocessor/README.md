# Coprocessor Specification

## Role

The coprocessor is the off-chain private execution system. It monitors `symVM`
events, reconstructs symbolic dependencies, resolves private computation, and
exposes resolved handle state to the `Coordinator`.

The coprocessor has two parts:

- a host service that monitors the chain, tracks handle state, and serves the
  internal API
- an enclave that performs private computation and produces attested outputs

## Responsibilities

- monitor the `symVM` event stream and associated chain metadata
- track issued handles, dependencies, and pending symbolic state
- schedule execution when the required handle inputs are available
- request re-encryption of input ciphertexts to an attested enclave key
- decrypt and evaluate symbolic operations inside the enclave
- package resolved outputs as `SystemCiphertextV1`
- expose handle status, `SystemCiphertextV1`, and execution receipts to the
  `Coordinator`

## Interfaces

### Coprocessor <- `symVM` / Chain

The coprocessor host:

- monitors the canonical `symVM` event stream
- reconstructs handle lineage and dependencies
- tracks which handles are pending, ready, or failed

### `Coordinator` -> Coprocessor

The coordinator uses the coprocessor to:

- request resolution of a handle
- fetch `SystemCiphertextV1`
- fetch execution receipts and failure status

### Coprocessor -> `MPC`

The coprocessor uses `MPC` to:

- transform `SystemCiphertextV1` inputs to `EnclaveCiphertextV1` for an
  attested enclave key
- request signatures or equivalent authorization artifacts when needed

## Detailed Specs

- [`./coprocessor-api.md`](./coprocessor-api.md)

## Trust Boundaries

- chain monitoring and request routing happen in the coprocessor host
- private computation happens inside the enclave
- the coprocessor does not receive raw decryption material from `MPC`
- handle results leave the enclave as `SystemCiphertextV1` plus execution
  receipts
