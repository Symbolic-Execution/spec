# `symVM` Operations

## Operation Model

Every `symVM` call returns a fresh handle.

In the base model:

- the initial handle types are `suint256` and `sbool`
- inputs remain valid after the call
- plaintext is not revealed
- `sbool` remains a handle, not a Solidity `bool`
- contracts express symbolic intent immediately and resolution happens later

## Operation Surface

### Arithmetic

- `add(suint256, suint256) -> suint256`
- `sub(suint256, suint256) -> suint256`

### Comparisons

- `eq(suint256, suint256) -> sbool`
- `lt(suint256, suint256) -> sbool`
- `lte(suint256, suint256) -> sbool`
- `gt(suint256, suint256) -> sbool`
- `gte(suint256, suint256) -> sbool`

### Boolean Operations

- `and(sbool, sbool) -> sbool`
- `or(sbool, sbool) -> sbool`
- `not(sbool) -> sbool`

### Selection

- `select(sbool, suint256, suint256) -> suint256`
- `select(sbool, sbool, sbool) -> sbool`

`select` is the private conditional-choice primitive of `symVM`.

It returns:

- the second input when the predicate is true
- the third input when the predicate is false

Selection rules:

- the first input is always `sbool`
- the second and third inputs have the same handle type
- the output type matches the second and third inputs
- the chosen branch is not observable to the contract at expression time

### Plaintext Conversion

- `fromPlaintext(uint256) -> suint256`
- `fromPlaintext(bool) -> sbool`

`fromPlaintext` turns a public constant into a handle. The value is public
on-chain. It is not a client-encrypted input.

The coprocessor later materializes `SystemCiphertextV1` for the resulting
handle so that the handle can participate in later symbolic operations.

Use `fromPlaintext` for reusable public constants such as:

```
suint256 zero = SYM.fromPlaintext(0);
suint256 transferValue = SYM.select(canTransfer, amount, zero);
```

## Authorization To Use Handles

`symVM` validates that the calling contract is authorized to use each input
handle before executing an operation.

A contract may use a handle in an operation if and only if:

- the handle was created by the same contract in the same domain, or
- the handle was explicitly allowed to the calling contract

A handle owner may manage operation-use grants with:

- `allow(handle_id, target)` — grants `target` persistent permission to use
  `handle_id` as an input to operations
- `revoke(handle_id, target)` — removes a previously granted persistent
  permission
- `allowTransient(handle_id, target)` — grants `target` permission to use
  `handle_id` within the current transaction only

Rules:

- only the creating contract may call `allow`, `revoke`, and `allowTransient`
- `allowTransient` lasts only for the current transaction
- grants authorize operation use only, not disclosure
- grants do not transfer ownership
- transient forwarding is single-hop because only the creating contract can
  grant again

## Invalid Calls

- wrong arity or wrong input types are invalid
- `select` requires an `sbool` predicate and the same type on both branches
- uninitialized handles are invalid inputs

## Not Included

This operation surface does not define:

- multiplication or division
- mixed public and symbolic arithmetic
- aggregate operations
- bitwise operations
