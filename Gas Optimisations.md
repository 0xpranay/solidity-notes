#### Read/Write costs : 

Becuase solidity operates in 32 bytes at once, read/write to an element less than 32 bytes **costs more** as **EVM needs to do extra OPs retrieve and pad extra bytes to the element in that 32 byte slot**.

Ironically, operations with **32 bytes** costs less as no extra operations. So **it is beneficial** to use reduced size elements like `uint8` **if you read them all at once, like a whole 32 byte slots at once as there isn't any padding operation involved **. So reading 4 `uint64` which are declared contiguosly is better **as there aren't any discard operations involved.** 

###### Solution :

Recall that structs and arrays start at a diff slot. So inside a struct declaring `uint128, uint128, uint256` is better optimised as just **2 slots** in storage.

But, `uint128, uint256, uint128` is worst due to **3 slots** as storage make a **new slot** if a variable can't fit.

#### Creating a function for encoding function selector : 

Instead of `addr.call(abi.encodeWithSignature("transfer(address,uint256)", 0xSomeAddress, 123))`, we can prevent computing the signature everytime and let a function do it like this,

```solidity
function getSelector(string calldata _func) external pure returns (bytes4) {
        return bytes4(keccak256(bytes(_func)));
    }
```

And use `addr.call(getSelector(funcName), 0xSomeAddress, 123))`. This saves a tiny amount of gas.