# Symbolic Execution Specifications

`Symbolic Execution` specifies a model for private, composable execution on
Ethereum.

The current design is informed by
[ERC-7984](https://eips.ethereum.org/EIPS/eip-7984): private values appear
on-chain as opaque references, compose across contracts, and are realized
asynchronously off-chain.

## Core Model

Private computation on Ethereum should look like a native contract primitive.

That means:

- private values appear on-chain as opaque handles rather than plaintext
- contracts can express operations over those values
- private state can compose across applications
- private execution can happen off-chain while the contract-facing abstraction
  remains stable

## Architecture Modules

The current minimal architecture has five pieces:

- `sym-client`: SDK that prepares private inputs, manages reader keys, and
  receives authorized outputs
- `symVM`: exposes private handles and symbolic operations on-chain
- `Coordinator`: off-chain control plane for authorization, async request
  tracking, and routing between services
- `Coprocessor`: resolves symbolic work off-chain
- `MPC`: provides threshold key custody, authorized ciphertext
  transformation, re-encryption, and related authorization flows

## High-Level Flow

```mermaid
sequenceDiagram
    autonumber
    box "User / SDK"
        participant U as "User"
        participant SC as "sym-client SDK"
    end
    box "Off-chain Control Plane"
        participant C as "Coordinator API"
    end
    box "On-chain"
        participant TK as "Smart Contract"
        participant SV as "symVM"
    end
    box "Off-chain Private Execution"
        participant CP as "Coprocessor (TEE Enclave)"
        participant M as "MPC"
    end

    U->>SC: Prepare private input
    SC->>C: GET /v1/config
    C-->>SC: Current public config and suite
    SC->>SC: Encrypt value as SystemCiphertextV1
    SC->>TK: Submit tx with encrypted value or handle request
    TK->>SV: Create handle and record symbolic operation
    SV-->>CP: Emit HandleImportedV1 / OperationRequestedV1
    CP->>CP: Reconstruct symbolic graph inside enclave
    CP->>CP: Generate enclave keypair and attestation
    CP->>M: Request authorized re-encryption to enclave key
    M-->>CP: EnclaveCiphertextV1
    Note over CP,M: MPC re-encrypts ciphertexts to the enclave key.
    CP->>CP: Decrypt with enclave private key and process computation
    CP->>CP: Materialize private state and bind result to handle
    CP-->>C: Publish status, receipts, and result references

    U->>SC: Request asynchronous disclosure for a handle
    SC->>C: POST /v1/disclosures {signed request, reader key}
    C-->>SC: 202 Accepted + request_id
    C->>C: Verify user signature and on-chain policy
    C->>CP: Resolve handle or fetch current SystemCiphertextV1
    CP-->>C: SystemCiphertextV1
    C->>M: Request authorized re-encryption to reader key
    M-->>C: ReaderCiphertextV1
    SC->>C: GET /v1/disclosures/{request_id}
    C-->>SC: ReaderCiphertextV1
    Note over SC,C: User disclosures are initiated through the Coordinator and remain encrypted to the reader key until sym-client decrypts locally.
    SC->>SC: Decrypt with reader secret
    SC-->>U: Return plaintext result
```

Execution split:

- `sym-client` encrypts user inputs before transactions are submitted on-chain.
- `symVM` records symbolic intent on-chain and emits the canonical event
  surface consumed by the `Coprocessor`.
- `Coordinator` is the off-chain entrypoint for user-facing async requests. It
  verifies signatures, checks policy, tracks request state, and routes work to
  the `Coprocessor` and `MPC`.
- private computation happens inside the coprocessor enclave.
- `MPC` handles threshold key operations such as re-encryption to an enclave
  key, re-encryption to a reader key, and signing.
- user reads are asynchronous: the user asks through `sym-client`, the
  `Coordinator` authorizes and tracks the request, `MPC` returns a
  reader-targeted ciphertext, and `sym-client` decrypts it locally.

## Scope

This specification currently focuses on:

- core terminology
- private values and symbolic operations
- how `sym-client`, `symVM`, `Coordinator`, `Coprocessor`, and `MPC` connect
- operation lifecycle from expression to materialization or disclosure
- permissions for reads and disclosure
- reusable application patterns built on top of the model

## Specs

- [`./sym-client/README.md`](./sym-client/README.md)
- [`./coordinator/README.md`](./coordinator/README.md)
- [`./coprocessor/README.md`](./coprocessor/README.md)
- [`./mpc/README.md`](./mpc/README.md)
- [`./symvm/README.md`](./symvm/README.md)
- [`./symvm/symvm-private-handles.md`](./symvm/symvm-private-handles.md)
- [`./symvm/symvm-operations.md`](./symvm/symvm-operations.md)
- [`./symvm/symvm-event-surface.md`](./symvm/symvm-event-surface.md)
- [`./symvm/symvm-operation-lifecycle.md`](./symvm/symvm-operation-lifecycle.md)
- [`./symvm/symvm-permissions-and-reads.md`](./symvm/symvm-permissions-and-reads.md)
