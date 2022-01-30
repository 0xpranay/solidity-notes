---
title : Solidity Gas Optimisations
---

[**← Back to Homepage**](https://0xpranay.github.io/solidity-notes/)

**Gas Optimisations** : This doc suggests some optimisations to save gas. Sources are mostly from **Solidity Docs**, **StackExchange** and more.

## Read/Write costs : 

Because solidity operates in 32 bytes at once, read/write to an element less than 32 bytes **costs more** as **EVM needs to do extra OPs retrieve and pad extra bytes to the element in that 32 byte slot**.

So, operations with **32 bytes** costs less as no extra operations are involved. So **it is beneficial** to use reduced size elements like `uint8` **if you read them all at once, like a whole 32 byte slots at once as there isn't any padding operation involved **. So reading 4 `uint64` which are declared contiguosly is better **as there aren't any discard operations involved.** 

##### Solution :

Recall that structs and arrays start at a diff slot. So inside a struct declaring `uint128, uint128, uint256` is better optimised as just **2 slots** in storage.

But, `uint128, uint256, uint128` is worst due to **3 slots** as storage make a **new slot** if a variable can't fit.

## Avoid calculating function selectors at runtime:

The compiler can precalculate a function selector at compilation time when the function signature is known.
Try to rely on this behavior whenever possible.

For example the following code prevents this optimization:
```solidity
function getSelector(bytes memory _func) returns (bytes4) {
    return bytes4(keccak256(_func));
}

function doStuff() {
    addr.call(abi.encodeWithSelector(getSelector("transfer(address,uint256)"), 0xSomeAddress, 123));
}
```
`getSelector()` will always run `keccak256()` on its argument because it cannot assume that it's always a constant.
With `abi.encodeWithSignature()`, on the other hand, the compiler is smart enough to generate more efficient code when you use a string literal:

```selector
addr.call(abi.encodeWithSignature("transfer(address,uint256)", 0xSomeAddress, 123));
```

If you inspect the resulting assembly, you'll notice that the value `0x6628616464726573732c75696e7429` (i.e. `"transfer(address,uint256)"`) is nowhere to be found and the result of the `keccak256()` call (`0xa9059cbb`) is hard-coded in it instead.

**Note**: Calculating selectors by hand is very error-prone and should be avoided.
A very common mistake, for example, is to insert a space after the comma, which will result in a completely different selector.
The example above does such a calculation on purpose, to illustrate a point, but in practice the language provides a more type-safe way to achieve the same result.
The `.selector` member:

```selector
addr.call(abi.encodeWithSelector(IERC20.transfer.selector, 0xSomeAddress, 123));
```

## **Use Events for storing something user related/temporary** : 

- Use Events to store per user actions or UI related data. Using storage is costly and creates unnecessary getters and overhead
- With events, something like TheGraph can be used to query them in an even powerful way than just storage view queries.

## **Use External functions when possible** : 

- External function's parameters can be declared `calldata` and will be fetched from `calldata` itself when referenced.
- For other types like `internal` and `public`, the parameters will be copied to memory by default which increases gas cost.

## **Revert strings** :

- Long strings in `require()` or `revert()` too increase gas cost. Use [custom errors](https://docs.soliditylang.org/en/latest/contracts.html#errors-and-the-revert-statement) instead.
- If you need large strings for some huge application, you can construct them on the UI side from the error type. You can also include additional parameters in the error, allowing your UI to insert them into the final message.
- For maximum savings you can set `debug.revertStrings` setting to `"strip"`, which will strip **all** revert strings from your code.
    Note, however, that this will make the failures of your contract very cryptic to users so it's not recommended.
    In fact, it's a good idea to have this set to `"debug"` in non-production builds of your contract because the default value strips messages from reverts generated automatically by the compiler making them hard to debug.

## Don't repeat yourself : 

- If you are computing a process always when required, it greatly increases gas cost. 
- Instead, write a function and let it do the task. This way, you can decrease bytecode size and save some gas.

## **Short Circuiting**:

- Imagine you have to check 2 conditions one after other to pass the check. You should check the lower cost/less complex check first instead of high cost check.
- Let `g(x)` be low cost and `h(x)` be high cost. Doing this saves some gas,
  - `g(x) && h(x)` as if `g(x)` fails, `h(x)` is never executed.
- Note that `require()` **does not** short-circuit, which matters if your message is not a constant and is calculated at runtime instead.
    This:
    ```solidity
    if (condition)
        revert(getMessage());
    ```
    is more efficient than:
    ```solidity
    require(condition, getMessage());
    ```
    because it will not run `getMessage()` when the `condition` is false.

## **Mappings are cheaper than arrays**:

- Because EVM store is a key value mapped store, mappings are cheaper for EVM than arrays
- Check **internals** section for more details.

## **bytes vs byte1[]**:

- Again, check **internals** section for a detailed explanation.
- TL;DR ? `bytes` has packing, `byte1[]` lacks packing in storage.

## **Limit external calls:**

- This might be or not be possible. But try to decrease them as much as possible.
- Every `external` call costs more gas than a simple internal call.

## **Make your contract upgradeable** : 

- Upgradeable contracts can have persistent storage while their code can be change or upgraded when a bug is discovered.
- This way, you don't need to copy all storage when creating a newer version of contract.

## Divide your code into multiple contracts:

- This is relevant only if the size of your bytecode is reaching the limit.
- By deploying different contracts, you can potentially deploy as much bytecode as you want.
