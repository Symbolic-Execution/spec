# `sym-client` API

## Scope

Define the client-side flows used by `sym-client`:

- reader-key management
- input encryption
- signed coordinator requests
- local decryption of reader-targeted ciphertexts

## Reader Keys

`sym-client` maintains a dedicated reader keypair for decryption.

```rust
pub struct ReaderKeyPair {
    pub public_key: [u8; 32],
    pub secret_key: [u8; 32],
}
```

Reader key rules:

- the reader keypair is separate from the Ethereum signing key
- one reader keypair is associated with one `sym-client` installation or device
- rotation creates a new keypair and a new `reader_id`
- `reader_id` is `keccak256(reader_pubkey)`

## Public Configuration

Before encrypting inputs or signing coordinator requests, `sym-client`
fetches `GET /v1/config` from the `Coordinator`.

That response provides:

- `chain_id`
- `domain_id`
- `mpc_key_id`
- `mpc_hpke_public_key`
- `reader_key_algorithm`
- `ciphertext_suite`
- `eip712_domain`

The response format is defined in
[`../coordinator/coordinator-api.md`](../coordinator/coordinator-api.md).

## Plaintext Encoding

The initial plaintext type surface is:

```rust
pub enum PlaintextValue {
    Suint256([u8; 32]),
    Sbool(bool),
}
```

Encoding rules:

- `Suint256` is encoded as a 32-byte big-endian byte string
- `Sbool(true)` is encoded as `0x01`
- `Sbool(false)` is encoded as `0x00`
- the encoded payload is wrapped in canonical CBOR before encryption

## Input Encryption

`sym-client` encrypts private inputs as `SystemCiphertextV1`.

Required inputs:

- `chain_id`
- `domain_id`
- `contract`
- `type_tag`
- `key_id`
- `mpc_hpke_public_key`
- plaintext encoded as canonical CBOR

Encryption flow:

1. fetch `GET /v1/config`
2. encode the plaintext as canonical CBOR
3. construct `SystemInputAadV1`
4. generate a random data-encryption key
5. encrypt the plaintext with `AES-256-GCM`
6. wrap the data-encryption key to `mpc_hpke_public_key` using `HPKE`
7. package the result as `SystemCiphertextV1`

`SystemInputAadV1` and `SystemCiphertextV1` are defined in
[`../mpc/mpc-api.md`](../mpc/mpc-api.md).

## Reader Registration

`sym-client` registers a reader key through
`PUT /v1/readers/{reader_id}` on the `Coordinator`.

Registration flow:

1. generate or load the reader keypair
2. derive `reader_id`
3. generate a fresh `nonce`
4. sign `RegisterReader` with the Ethereum signing key using the coordinator
   EIP-712 domain
5. call `PUT /v1/readers/{reader_id}`

`RegisterReader` and the request format are defined in
[`../coordinator/coordinator-api.md`](../coordinator/coordinator-api.md).

## Disclosure Requests

`sym-client` requests disclosure through `POST /v1/disclosures`.

Disclosure flow:

1. generate a fresh `nonce`
2. sign `DisclosureRequest` with the Ethereum signing key using the coordinator
   EIP-712 domain
3. call `POST /v1/disclosures`
4. if the response is `Pending`, poll `GET /v1/disclosures/{request_id}`
5. when the response is `Ready`, decrypt `ReaderCiphertextV1` locally

`DisclosureRequest`, `DisclosureStatus`, and the response formats are defined in
[`../coordinator/coordinator-api.md`](../coordinator/coordinator-api.md).

## Local Decryption

`sym-client` decrypts `ReaderCiphertextV1` with the reader secret key.

Decryption flow:

1. use the reader secret key to open `ReaderCiphertextV1`
2. parse and verify `ReaderAadV1`
3. decode the canonical-CBOR plaintext
4. return the typed plaintext value

`ReaderCiphertextV1` and `ReaderAadV1` are defined in
[`../mpc/mpc-api.md`](../mpc/mpc-api.md).

## Nonces And Expiry

- `sym-client` generates a fresh random nonce for each signed coordinator
  request
- registration and disclosure requests include `expiry`
- expired requests are not retried with the same `(controller, nonce)`
