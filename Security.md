## 1. Ether Balance

Never try to calculate the balance of the contract using manual counters. A vulnerable approach is that everytime the contract receives ether, based on the functions implemented, **people update state variables as a counter/balance received**. This is wrong as `coinbase` transactions and `selfdestruct` destination transactions can **Force Ether in!**. They do not trigger any functions and don't care if you have `payable` functions or `fallback` functions. **The ether balance increases**.

##### Solution : 

Always use `balance(this)` to get the balance of contract. This also is **NOT** good. See below.

## 2. Outliers : 

EVM defines `0**0` as `1`.

Because of wrapping, `assert(x == -x)` becomes **true** when `x = type(int).min` as positive overflows and becomes `int.MIN` again. 

Right and left shift results are capped at the type extremes.

Division by zero and Modulo by zero can't be suppresed with even `unchecked{}` as they are `panic` errors.

Calling a function after using `delete` on it causes a `panic` error.

## 3. Overflow/Underflow

Prior to 0.8, solidity used to underflow/overflow without any errors.

After 0.8, an error is thrown. More better, use something like **SafeMath library**

## 4. Selfdestruct : 

A contract can self destruct itself and force it's ether into a victim contract. This makes the victim **not aware of balance increase** as **no function, not even fallback/receive is called for a self destruct**. So using `address(this).balance` might break **when someone does the selfdestruct attack**.

Contrast this with above, we have 2 choices. 

- **If you want to handle legit users depositing ETH, use the `balance` state variable**
- **If you also want to handle force sends, then use `this.balance`**
- My take : **Go with the `balance` state variable and also include a function that can take the difference `this.balance - balance` and call deposit function again**

## 5. The blockchain is transparent :

- All info about state variables can be accessed with a web3 client/a node. So even though the **visibility** is just **private**, anyone with node access can see the contents of the data.

## 6. Randomness:

- The blockchain in it's intended way to work is **Deterministic**. Dont' use `block.timestamp` and `block.blockhash` as source of randomness.
- Use something like chainlink VRF. Some even suggest using BTC block hashes as block reward is higher than ETH so miners can be more trusted.

##  7.`require` is also treated as a function : 

- The `require` function is evaluated just as any other function. This means that all arguments are evaluated before the function itself is executed. In particular, in `require(condition, f())` the function `f` is executed even if `condition` is true.
- So don't be fooled by some honeypot contract's code saying `require(obviouslyTrue, malicoiusCode())`. 

## 8. Gas DDoS

- Looping over dynamic arrays is dangerous. The array may grow so much in future that the operation of the array's deletion/iteration **becomes more than block gas limit making the contract unusable**
- Hence, avoid such opeartions or **divide the operations into different blocks**. Like empty a part of array and store the indice in storage for next block.

## 9. DAG (C3) Abuse : 

- Read the C3 Linearization in `general notes` if you didn't. If overlooked, C3 Linearization might expose a potential flaw.
- The `super.func()` might yield unexpected results if the linearization decided inheritance order different from what the user expected. See [Here](https://pdaian.com/blog/solidity-anti-patterns-fun-with-inheritance-dag-abuse/).

## 10. Dynamic array/mapping backdoor : 

- [This](https://github.com/Arachnid/uscc/tree/master/submissions-2017/doughoyte) explains it better than anything. 
- TL;DR? Simply, a contract's owner can maliciously manipulate some storage variables if they don't implement strict dynamic array length checks. 

## 11. Writing critical code yourself : 

- **DO NOT EVER WRITE SOMETHING LIKE ** `ecrecover` **YOURSELF!**. Use industry standard libraries like ones at openzeppelin. 

## 12. Replay Attacks : 

- Often we users to sign messages off chain and then send them on chain later. **This can lead to replay attacks**
- So, to avoid some attacker replaying the sign by **storing the hash of processed transactions**
- And also to avoid replay between different contracts, **include the contract address and chain id too**. 

## 13. Timestamps : 

- There's little error here but worth learning. The `block.timestamp` should not be used to precisely calculate time. It can only be used for large values like days/weeks and above.
- `block.number` to estimate time too is not exactly correct. There might be a change in ethereum's 14 second block time and your hardcoded value might be off a bit too much. Consensys says if you can tolerate a 15 second diff, safe enough to use.
- DO NOT ever ever use `block.timestamp` as source of randomness. 

