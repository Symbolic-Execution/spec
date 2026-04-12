# `symVM` Permissions And Reads

## Goal

Define how `symVM` separates symbolic handle possession from authorization to
read underlying private values.

This document focuses on the minimum disclosure model needed for the first
confidential token-oriented specs.

## Working Model

In `symVM`, possession of a handle is not the same thing as permission to read
the plaintext value behind that handle.

Reads are explicit actions governed by policy.

In the current working model:

- handles may circulate across contracts without automatically granting read
  access
- read access must be authorized explicitly
- disclosure should target an authorized reader rather than publishing plaintext
  on-chain by default
- read requests are asynchronous, like the rest of the private computation model

The intended read model is:

- private user reads are fulfilled privately to the authorized reader
- contract-visible disclosure is left out of the base spec unless a higher-level
  ERC requires it

## Requirements

### Explicit Authorization

The system must not treat handle possession alone as sufficient authorization
for plaintext disclosure.

### Reader-Targeted Disclosure

The default read model should reveal plaintext to an authorized reader, not to
the whole chain.

### Policy-Separable From Handle Shape

The permission model should be able to evolve without changing the basic handle
types.

### Composable Across Applications

Applications must be able to build reusable read and delegation patterns on top
of the same underlying `symVM` model.

### No Accidental Public Declassification

The contract model should make private-to-public disclosure explicit rather than
letting it happen as a side effect of ordinary reads.

## Reads Are Disclosure Requests

The current working choice is to model reads as explicit disclosure requests.

That means a read is not "loading" a private value into Solidity. It is a
request for the system to disclose a handle's underlying value to an authorized
recipient under an explicit policy.

## Recipient-Targeted Disclosure

The current working model is to make recipient-targeted disclosure the default
read model.

In practice, that means the resolved value should be delivered to a specific
authorized reader or reader key, rather than emitted as plaintext on-chain.

This preserves the privacy model more naturally and matches the intended use
case of confidential balances and confidential application state.

## Current Fulfillment Mode

The current working model is to specify only:

- private reader disclosure, which is returned off-chain to the authorized
  reader after validation

If a higher-level ERC needs contract-visible disclosure, it should define that
flow separately.
