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

Read model:

- handles may circulate across contracts without automatically granting read
  access
- read access must be authorized explicitly
- disclosure targets an authorized reader rather than publishing plaintext
  on-chain by default
- read requests are asynchronous, like the rest of the private computation model
- private reader requests are submitted off-chain to the `Coordinator`, not to
  `symVM`, in the base spec

Read fulfillment:

- private user reads are fulfilled privately to the authorized reader
- contract-visible disclosure is left out of the base spec unless a higher-level
  ERC requires it

Disclosure authorization uses the following model:

- the `Coordinator` accepts an EIP-712 signed disclosure request from the
  handle's controlling account, as defined by the higher-level standard
- no persistent base-spec approval registry is required

## Requirements

### Explicit Authorization

The system must not treat handle possession alone as sufficient authorization
for plaintext disclosure.

### Reader-Targeted Disclosure

The default read model reveals plaintext to an authorized reader, not to
the whole chain.

### Policy-Separable From Handle Shape

The permission model can evolve without changing the basic handle
types.

### Composable Across Applications

Applications must be able to build reusable read and delegation patterns on top
of the same underlying `symVM` model.

### No Accidental Public Declassification

The contract model makes private-to-public disclosure explicit rather than
letting it happen as a side effect of ordinary reads.

## Reads Are Off-Chain Disclosure Requests

Reads are explicit off-chain disclosure requests handled by the `Coordinator`.

That means a read is not "loading" a private value into Solidity. It is a
request for the system to disclose a handle's underlying value to an authorized
recipient under an explicit policy.

`symVM` provides the on-chain handles and permission surface that the
`Coordinator`, coprocessor, and `MPC` consult, but the request itself is not a
base `symVM` operation.

The `Coordinator` checks the handle's controlling account on-chain and verifies
a matching signed request.

## Recipient-Targeted Disclosure

Recipient-targeted disclosure is the default read model.

The resolved value is delivered to a specific authorized reader or reader key,
rather than emitted as plaintext on-chain.

## Fulfillment Mode

This spec defines only:

- private reader disclosure, which is returned off-chain to the authorized
  reader after validation by the `Coordinator` and backend services

A higher-level ERC that needs contract-visible disclosure defines that flow
separately.
