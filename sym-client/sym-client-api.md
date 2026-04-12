# `sym-client` API

## Scope

Define the client-side data types, encryption flow, reader-key lifecycle, and
disclosure flow used by `sym-client`.

## Serialization

Coordinator requests and responses use JSON.

Binary values are encoded as lowercase `0x`-prefixed hex strings.

Private plaintext payloads are encoded as canonical CBOR before encryption.

## Core Types

```rust
pub type Address = [u8; 20];
pub type Bytes32 = [u8; 32];
pub type UnixSeconds = u64;

pub struct ReaderId(pub Bytes32);
pub struct RequestId(pub Bytes32);
pub struct HandleId(pub Bytes32);
pub struct Nonce(pub Bytes32);

pub struct X25519PublicKey(pub [u8; 32]);
pub struct X25519SecretKey(pub [u8; 32]);
pub struct EthereumSignature(pub [u8; 65]);
```

## Reader Keys

`sym-client` maintains a dedicated reader keypair for decryption.

```rust
pub struct ReaderKeyPair {
    pub public_key: X25519PublicKey,
    pub secret_key: X25519SecretKey,
}
```

`reader_id` is defined as:

```rust
pub fn reader_id(reader_pubkey: X25519PublicKey) -> ReaderId {
    ReaderId(keccak256(reader_pubkey.0))
}
```

Reader key rules:

- a reader keypair is separate from the Ethereum signing key
- one reader keypair is associated with one `sym-client` installation or device
- rotation creates a new keypair and a new `reader_id`
- `sym-client` uses the matching reader secret key to decrypt
  `ReaderCiphertextV1`

## Public Configuration

Before encrypting inputs or signing coordinator requests, `sym-client`
fetches:

```rust
pub struct CoordinatorConfig {
    pub version: u16,
    pub chain_id: u64,
    pub domain_id: Bytes32,
    pub mpc_key_id: Bytes32,
    pub mpc_hpke_public_key: Bytes32,
    pub reader_key_algorithm: ReaderKeyAlgorithm,
    pub ciphertext_suite: CiphertextSuite,
    pub eip712_domain: Eip712Domain,
}

pub enum ReaderKeyAlgorithm {
    X25519,
}

pub enum CiphertextSuite {
    HpkeX25519HkdfSha256Aes256Gcm,
}

pub struct Eip712Domain {
    pub name: String,
    pub version: String,
    pub chain_id: u64,
    pub salt: Bytes32,
}
```

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

```rust
pub struct EncryptInputRequest {
    pub chain_id: u64,
    pub domain_id: Bytes32,
    pub contract: Address,
    pub type_tag: String,
    pub key_id: Bytes32,
    pub mpc_hpke_public_key: Bytes32,
    pub plaintext: PlaintextValue,
}

pub struct EncryptInputResponse {
    pub ciphertext: SystemCiphertextV1,
}
```

Encryption flow:

1. fetch `CoordinatorConfig`
2. encode the plaintext as canonical CBOR
3. construct the `aad` for the target contract and type
4. generate a random data-encryption key
5. encrypt the plaintext with `AES-256-GCM`
6. wrap the data-encryption key to `mpc_hpke_public_key` using `HPKE`
7. package the result as `SystemCiphertextV1`

`aad` binds the ciphertext to protocol context such as:

- `chain_id`
- `domain_id`
- `contract`
- `type_tag`
- `key_id`

## Reader Registration

`sym-client` registers a reader key by signing a coordinator request with the
user's Ethereum signing key.

```rust
pub struct RegisterReaderRequest {
    pub controller: Address,
    pub reader_pubkey: X25519PublicKey,
    pub nonce: Nonce,
    pub expiry: UnixSeconds,
}

pub struct RegisterReader {
    pub controller: Address,
    pub reader_id: ReaderId,
    pub nonce: Nonce,
    pub expiry: UnixSeconds,
}
```

Registration flow:

1. generate or load the reader keypair
2. derive `reader_id`
3. generate a fresh `nonce`
4. sign `RegisterReader` with the Ethereum signing key using the coordinator
   EIP-712 domain
5. call `PUT /v1/readers/{reader_id}`

## Disclosure Requests

`sym-client` requests disclosure by signing a coordinator request with the
user's Ethereum signing key.

```rust
pub struct DisclosureRequestInput {
    pub contract: Address,
    pub handle_id: HandleId,
    pub reader_id: ReaderId,
    pub nonce: Nonce,
    pub expiry: UnixSeconds,
}

pub struct DisclosureRequest {
    pub contract: Address,
    pub handle_id: HandleId,
    pub reader_id: ReaderId,
    pub nonce: Nonce,
    pub expiry: UnixSeconds,
}

pub enum DisclosureStatus {
    Pending,
    Ready,
    Failed,
    Expired,
}
```

Disclosure flow:

1. generate a fresh `nonce`
2. sign `DisclosureRequest` with the Ethereum signing key using the coordinator
   EIP-712 domain
3. call `POST /v1/disclosures`
4. if the response is `Pending`, poll `GET /v1/disclosures/{request_id}`
5. when the response is `Ready`, decrypt `ReaderCiphertextV1` locally

## Local Decryption

`sym-client` decrypts `ReaderCiphertextV1` with the reader secret key.

```rust
pub struct DecryptReaderCiphertextRequest {
    pub reader_secret_key: X25519SecretKey,
    pub ciphertext: ReaderCiphertextV1,
}

pub struct DecryptReaderCiphertextResponse {
    pub plaintext: PlaintextValue,
}
```

Decryption flow:

1. use the reader secret key to open `ReaderCiphertextV1`
2. verify the `aad` bindings
3. decode the canonical-CBOR plaintext
4. return the typed plaintext value

## Nonces And Expiry

- `sym-client` generates a fresh random `Nonce` for each signed coordinator
  request
- registration and disclosure requests include `expiry`
- expired requests are not retried with the same `(controller, nonce)`

## Ciphertext Types

`sym-client` uses these ciphertext envelopes:

```rust
pub struct SystemCiphertextV1 {
    pub key_id: Bytes32,
    pub enc: Vec<u8>,
    pub wrapped_key: Vec<u8>,
    pub nonce: [u8; 12],
    pub ciphertext: Vec<u8>,
    pub aad: Vec<u8>,
}

pub struct ReaderCiphertextV1 {
    pub key_id: Bytes32,
    pub enc: Vec<u8>,
    pub ciphertext: Vec<u8>,
    pub aad: Vec<u8>,
}
```
