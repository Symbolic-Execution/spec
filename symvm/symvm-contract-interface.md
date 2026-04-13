# `symVM` Contract Interface

## Model

`ISymVM` is the low-level on-chain interface.
`SYM` is the typed Solidity library that most contracts should use.

An integrating contract stores the `symVM` address once. `SYM` reads that
address on every call.

## Configuration

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

Write this slot during construction for direct deployments or during
initialization for proxy deployments.

## `ISymVM` Interface

```solidity
interface ISymVM {
    function importCiphertext(
        uint8 handleType,
        bytes calldata systemCiphertext
    ) external returns (bytes32 handleId);

    function fromPlaintext(
        uint8 handleType,
        bytes32 value
    ) external returns (bytes32 handleId);

    function add(bytes32 a, bytes32 b) external returns (bytes32);
    function sub(bytes32 a, bytes32 b) external returns (bytes32);

    function eq(bytes32 a, bytes32 b) external returns (bytes32);
    function lt(bytes32 a, bytes32 b) external returns (bytes32);
    function lte(bytes32 a, bytes32 b) external returns (bytes32);
    function gt(bytes32 a, bytes32 b) external returns (bytes32);
    function gte(bytes32 a, bytes32 b) external returns (bytes32);

    function and_(bytes32 a, bytes32 b) external returns (bytes32);
    function or_(bytes32 a, bytes32 b) external returns (bytes32);
    function not_(bytes32 a) external returns (bytes32);

    function select(
        bytes32 pred,
        bytes32 whenTrue,
        bytes32 whenFalse
    ) external returns (bytes32);

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

`SYM` provides the typed developer-facing API. Each function unwraps typed
handles, calls `ISymVM`, and wraps the returned `bytes32`.

```solidity
library SYM {
    function importCiphertext(
        bytes calldata systemCiphertext
    ) internal returns (suint256);

    function importCiphertextBool(
        bytes calldata systemCiphertext
    ) internal returns (sbool);

    function fromPlaintext(uint256 value) internal returns (suint256);
    function fromPlaintext(bool value) internal returns (sbool);

    function add(suint256 a, suint256 b) internal returns (suint256);
    function sub(suint256 a, suint256 b) internal returns (suint256);

    function eq(suint256 a, suint256 b) internal returns (sbool);
    function lt(suint256 a, suint256 b) internal returns (sbool);
    function lte(suint256 a, suint256 b) internal returns (sbool);
    function gt(suint256 a, suint256 b) internal returns (sbool);
    function gte(suint256 a, suint256 b) internal returns (sbool);

    function and_(sbool a, sbool b) internal returns (sbool);
    function or_(sbool a, sbool b) internal returns (sbool);
    function not_(sbool a) internal returns (sbool);

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

Separate import functions are needed because Solidity cannot overload on return
type alone.

Use `fromPlaintext` to build public constants such as `0` or `false`. Do not
rely on `bytes32(0)`, which is an uninitialized handle.

## Handle Type IDs

`ISymVM` uses a `uint8` discriminant to identify handle types in untyped
calls:

```solidity
uint8 constant SUINT256 = 1;
uint8 constant SBOOL = 2;
```

These values match the `HandleType` enum defined in the event surface.

## Ownership And Grants

When a contract calls `ISymVM`, `msg.sender` becomes the creating contract of
the output handle.

- the creating contract may use its own handles without extra grants
- `allow` and `allowTransient` authorize other contracts to use a handle in
  operations
- disclosure is handled separately through the coordinator flow

## Example

This example imports two encrypted values and returns the smaller one without
revealing either plaintext:

```solidity
import {SYM, suint256, sbool} from "symvm/SYM.sol";

contract Example {
    function min(
        bytes calldata encryptedA,
        bytes calldata encryptedB
    ) external returns (suint256) {
        suint256 a = SYM.importCiphertext(encryptedA);
        suint256 b = SYM.importCiphertext(encryptedB);
        sbool aLteB = SYM.lte(a, b);
        return SYM.select(aLteB, a, b);
    }
}
```

This example is illustrative. Higher-level token or disclosure standards are
defined separately.
