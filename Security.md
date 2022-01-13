## Stack Swap

EVM allows an opcode that let's you swap last 16th value. If a function invokes other function, the stack might propagate and the inner function may access the caller's variables. (*Self idea, needs reference to make sure*).

##### Solution : 

---?

## Ether Balance

Never try to calculate the balance of the contract using manual counters. A vulnerable approach is that everytime the contract receives ether, based on the functions implemented, **people update state variables as a counter/balance received**. This is wrong as `coinbase` transactions and `selfdestruct` destination transactions can **Force Ether in!**. They do not trigger any functions and don't care if you have `payable` functions or `fallback` functions. **The ether balance increases**.

##### Solution : 

Always use `balance(this)` to get the balance of contract. This also is **NOT** good. See below.

## Outliers : 

EVM defines `0**0` as `1`.

Because of wrapping, `assert(x == -x)` becomes **true** when `x = type(int).min` as positive overflows and becomes `int.MIN` again. 

Right and left shift results are capped at the type extremes.

Division by zero and Modulo by zero can't be suppresed with even `unchecked{}` as they are `panic` errors.

Calling a function after using `delete` on it causes a `panic` error.

### Overflow/Underflow

Prior to 0.8, solidity used to underflow/overflow without any errors.

After 0.8, an error is thrown. More better, use something like **SafeMath library**

### Selfdestruct : 

A contract can self destruct itself and force it's ether into a victim contract. This makes the victim **not aware of balance increase** as **no function, not even fallback/receive is called for a self destruct**. So using `address(this).balance` might break **when someone does the selfdestruct attack**.

Contrast this with above, we have 2 choices. 

- **If you want to handle legit users depositing ETH, use the `balance` state variable**
- **If you also want to handle force sends, then use `this.balance`**
- My take : **Go with the `balance` state variable and also include a function that can take the difference `this.balance - balance` and call deposit function again**

### The blockchain is transparent :

- All info about state variables can be accessed with a web3 client/a node. So even though the **visibility** is just **private**, anyone with node access can see the contents of the data.

### Randomness:

- The blockchain in it's intended way to work is **Deterministic**. Dont' use `block.timestamp` and `block.blockhash` as source of randomness.
- Use something like chainlink VRF.

#### `require` is also treated as a function : 

- The `require` function is evaluated just as any other function. This means that all arguments are evaluated before the function itself is executed. In particular, in `require(condition, f())` the function `f` is executed even if `condition` is true.
- So don't be fooled by some honeypot contract's code saying `require(obviouslyTrue, malicoiusCode())`. 

