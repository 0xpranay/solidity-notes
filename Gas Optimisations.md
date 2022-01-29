---
title : Solidity Gas Optimisations
---

[**← Back to Homepage**](https://0xpranay.github.io/solidity-notes/)

**Gas Optimisations** : This doc suggests some optimisations to save gas. Sources are mostly from **Solidity Docs**, **StackExchange** and more.

## Read/Write costs : 

Becuase solidity operates in 32 bytes at once, read/write to an element less than 32 bytes **costs more** as **EVM needs to do extra OPs retrieve and pad extra bytes to the element in that 32 byte slot**.

So, operations with **32 bytes** costs less as no extra operations are involved. So **it is beneficial** to use reduced size elements like `uint8` **if you read them all at once, like a whole 32 byte slots at once as there isn't any padding operation involved **. So reading 4 `uint64` which are declared contiguosly is better **as there aren't any discard operations involved.** 

##### Solution :

Recall that structs and arrays start at a diff slot. So inside a struct declaring `uint128, uint128, uint256` is better optimised as just **2 slots** in storage.

But, `uint128, uint256, uint128` is worst due to **3 slots** as storage make a **new slot** if a variable can't fit.

## Creating a function for encoding function selector : 

Instead of `addr.call(abi.encodeWithSignature("transfer(address,uint256)", 0xSomeAddress, 123))`, we can prevent computing the signature everytime and let a function do it like this,

```solidity
function getSelector(string calldata _func) external pure returns (bytes4) {
        return bytes4(keccak256(bytes(_func)));
    }
```

And use `addr.call(getSelector(funcName), 0xSomeAddress, 123))`. This saves a tiny amount of gas.



- **Constants** don't occupy any storage. They're always **replaced when needed in the code like a macro**. 

## **Use Events for storing something user related/temporary** : 

- Use Events to store per user actions or UI related data. Using storage is costly and creates unnecessary getters and overhead
- With events, something like TheGraph can be used to query them in an even powerful way than just storage view queries.

## **Use External functions when possible** : 

- External function's parameters can be declared `calldata` and will be fetched from `calldata` itself when referenced.
- For other types like `internal` and `public`, the parameters will be copied to memory by default which increases gas cost.

## **Require strings** :

- Long require strings too increase gas cost. Use smaller strings.
- If you need large strings for some huge application, you can use some error codes as require strings and then maybe decode them on UI side, bypassing long string gas cost.

## Don't repeat yourself : 

- If you are computing a process always when required, it greatly increases gas cost. 
- Instead, write a function and let it do the task. This way, you can decrease bytecode size and save some gas.

## **Short Circuiting**:

- Imagine you have to check 2 conditions one after other to pass the check. You should check the lower cost/less complex check first instead of high cost check.
- Let `g(x)` be low cost and `h(x)` be high cost. Doing this saves some gas,
  - `g(x) && h(x)` as if `g(x)` fails, `h(x)` is never executed.

## **Mappings are cheper than arrays** : 

- Becuase EVM store is a key value mapped store, mappings are cheaper for EVM than arrays
- Check **internals** section for more details.

## **Bytes32 vs byte1[]** : 

- Again, check **internals** section for a detailed explanation.
- TL;DR ? `bytes32` has packing, `byte1[]` lacks packing in storage.

## **Limit external calls:**

- This might be or not be possible. But try to decrease them as much as possible.
- Every `external` call costs more gas than a simple internal call.

## **Make your contract upgradeable** : 

- Upgradeable contracts can have persistent storage while their code can be change or upgraded when a bug is discovered.
- This way, you don't need to copy all storage when creating a newer version of contract.

## Divide your code into multiple contracts:

- This is relevant only if the size of your bytecode is reaching the limit.
- By deploying different contracts, you can potentially deploy as much bytecode as you want.