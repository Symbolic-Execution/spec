# `symVM` Contract Interface

## Scope

Define the on-chain Solidity interface that contracts use to interact with
`symVM`. This is the integration surface between application contracts and
the symbolic execution system.

## Architecture

The `symVM` interface consists of:

- `ISymVM` — the on-chain contract interface for handle creation, symbolic
  operations, and authorization
- `SYM` — a Solidity library that wraps `ISymVM` calls with typed handle
  arguments and return values

Contracts import `SYM` and call its functions. `SYM` unwraps typed handles
to `bytes32`, dispatches to the `ISymVM` contract, and wraps the returned
`bytes32` as the correct handle type.

## Configuration

A contract that uses `symVM` must know the address of the `ISymVM` contract.

```solidity
struct SymVMConfig {
    address symvm;
}
```

The configuration is set once during initialization. For non-upgradeable
contracts this happens in the constructor. For proxy patterns this happens
in the initializer function. The `SYM` library reads the stored address for
every call.

The storage location uses a fixed slot to avoid collisions with contract
state and to work with both direct deployment and proxy patterns:

```solidity
bytes32 constant SYM_CONFIG_SLOT =
    keccak256("symbolic.symvm.config.v1");
```

## `ISymVM` Interface

```solidity
interface ISymVM {
    // Handle creation
    function importCiphertext(
        uint8 handleType,
        bytes calldata systemCiphertext
    ) external returns (bytes32 handleId);

    function fromPlaintext(
        uint8 handleType,
        bytes32 value
    ) external returns (bytes32 handleId);

    // Arithmetic
    function add(bytes32 a, bytes32 b) external returns (bytes32);
    function sub(bytes32 a, bytes32 b) external returns (bytes32);

    // Comparisons
    function eq(bytes32 a, bytes32 b) external returns (bytes32);
    function lt(bytes32 a, bytes32 b) external returns (bytes32);
    function lte(bytes32 a, bytes32 b) external returns (bytes32);
    function gt(bytes32 a, bytes32 b) external returns (bytes32);
    function gte(bytes32 a, bytes32 b) external returns (bytes32);

    // Boolean operations
    function and_(bytes32 a, bytes32 b) external returns (bytes32);
    function or_(bytes32 a, bytes32 b) external returns (bytes32);
    function not_(bytes32 a) external returns (bytes32);

    // Selection
    function select(
        bytes32 pred,
        bytes32 whenTrue,
        bytes32 whenFalse
    ) external returns (bytes32);

    // Authorization
    function allow(bytes32 handleId, address target) external;
    function revoke(bytes32 handleId, address target) external;
    function allowTransient(bytes32 handleId, address target) external;
    function isAllowed(
        bytes32 handleId,
        address account
    ) external view returns (bool);
}
```

Boolean operation names use a trailing underscore (`and_`, `or_`, `not_`)
to avoid collisions with Solidity reserved words.

## `SYM` Library

The `SYM` library provides the typed developer-facing API. Every function
unwraps its handle arguments, calls `ISymVM`, and wraps the result.

```solidity
library SYM {
    // Handle creation
    function importCiphertext(
        bytes calldata systemCiphertext
    ) internal returns (suint256);

    function importCiphertextBool(
        bytes calldata systemCiphertext
    ) internal returns (sbool);

    function fromPlaintext(uint256 value) internal returns (suint256);
    function fromPlaintext(bool value) internal returns (sbool);

    // Arithmetic
    function add(suint256 a, suint256 b) internal returns (suint256);
    function sub(suint256 a, suint256 b) internal returns (suint256);

    // Comparisons
    function eq(suint256 a, suint256 b) internal returns (sbool);
    function lt(suint256 a, suint256 b) internal returns (sbool);
    function lte(suint256 a, suint256 b) internal returns (sbool);
    function gt(suint256 a, suint256 b) internal returns (sbool);
    function gte(suint256 a, suint256 b) internal returns (sbool);

    // Boolean operations
    function and_(sbool a, sbool b) internal returns (sbool);
    function or_(sbool a, sbool b) internal returns (sbool);
    function not_(sbool a) internal returns (sbool);

    // Selection
    function select(
        sbool pred,
        suint256 whenTrue,
        suint256 whenFalse
    ) internal returns (suint256);

    function select(
        sbool pred,
        sbool whenTrue,
        sbool whenFalse
    ) internal returns (sbool);

    // Authorization
    function allow(suint256 handle, address target) internal;
    function allow(sbool handle, address target) internal;
    function revoke(suint256 handle, address target) internal;
    function revoke(sbool handle, address target) internal;
    function allowTransient(suint256 handle, address target) internal;
    function allowTransient(sbool handle, address target) internal;
    function isAllowed(suint256 handle, address account)
        internal view returns (bool);
    function isAllowed(sbool handle, address account)
        internal view returns (bool);
}
```

## Handle Type Discriminant

`ISymVM` uses a `uint8` discriminant to identify handle types in untyped
calls:

```solidity
uint8 constant SUINT256 = 1;
uint8 constant SBOOL = 2;
```

These values match the `HandleType` enum defined in the event surface.

## Derived Handle Ownership

When a contract calls an `ISymVM` operation, the `msg.sender` is the
calling contract. `symVM` records that contract as the creating contract of
the output handle.

For derived handles (from operations like `add`, `select`, etc.):

- the calling contract is the owner
- the calling contract is implicitly authorized to use the handle
- no explicit `allow` call is needed for the contract to use its own
  handles in subsequent operations

## Interaction Pattern

A typical contract interaction:

```solidity
import {SYM, suint256, sbool} from "symvm/SYM.sol";

contract ConfidentialToken {
    mapping(address => suint256) private balances;

    function transfer(address to, bytes calldata encryptedAmount) external {
        suint256 amount = SYM.importCiphertext(encryptedAmount);
        suint256 zero = SYM.fromPlaintext(0);

        sbool canTransfer = SYM.lte(amount, balances[msg.sender]);
        suint256 transferValue = SYM.select(canTransfer, amount, zero);

        suint256 newSenderBalance = SYM.sub(balances[msg.sender], transferValue);
        suint256 newReceiverBalance = SYM.add(balances[to], transferValue);

        balances[msg.sender] = newSenderBalance;
        balances[to] = newReceiverBalance;

        // Allow users to request disclosure of their own balance
        SYM.allow(newSenderBalance, msg.sender);
        SYM.allow(newReceiverBalance, to);
    }
}
```

This example is illustrative. The full confidential token standard is
defined separately.
