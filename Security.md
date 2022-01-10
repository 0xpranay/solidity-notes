#### Stack Swap

EVM allows an opcode that let's you swap last 16th value. If a function invokes other function, the stack might propagate and the inner function may access the caller's variables. (*Self idea, needs reference to make sure*).

##### Solution : 

---?

#### Ether Balance

Never try to calculate the balance of the contract using manual counters. A vulnerable approach is that everytime the contract receives ether, based on the functions implemented, **people update state variables as a counter/balance received**. This is wrong as `coinbase` transactions and `selfdestruct` destination transactions can **Force Ether in!**. They do not trigger any functions and don't care if you have `payable` functions or `fallback` functions. **The ether balance increases**.

##### Solution : 

Always use `balance(this)` to get the balance of contract.

#### Outliers : 

EVM defines `0**0` as `1`.

Because of wrapping, `assert(x == -x)` becomes **true** when `x = type(int).min` as positive overflows and becomes `int.MIN` again. 

Right and left shift results are capped at the type extremes.

Division by zero and Modulo by zero can't be suppresed with even `unchecked{}` as they are `panic` errors.

Calling a function after using `delete` on it causes a `panic` error.



