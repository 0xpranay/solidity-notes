**Security** : This doc helps readers know about security vulnerabilities in smart contracts and also developer practices in the end. This doc **by no means covers all the vulnerabilities** as smart contract security is an evergoing learning path.

## 1. Ether Balance

Never try to calculate the balance of the contract using manual counters. A vulnerable approach is that everytime the contract receives ether, based on the functions implemented, **people update state variables as a counter/balance received**. This is wrong as `coinbase` transactions and `selfdestruct` destination transactions can **Force Ether in!**. They do not trigger any functions and don't care if you have `payable` functions or `fallback` functions. **The ether balance increases**. Check out [GridLock's Bug](https://medium.com/@nmcl/gridlock-a-smart-contract-bug-73b8310608a9).

#### Solution : 

Always use `balance(this)` to get the balance of contract. This also is **NOT** good. See below.

## 2. Outliers : 

EVM defines `0**0` as `1`.

Because of wrapping, `assert(x == -x)` becomes **true** when `x = type(int).min` as positive overflows and becomes `int.MIN` again. 

Right and left shift results are capped at the type extremes.

Division by zero and Modulo by zero can't be suppresed with even `unchecked{}` as they are `panic` errors.

Calling a function after using `delete` on it causes a `panic` error.

## 3. Overflow/Underflow

Prior to 0.8, solidity used to underflow/overflow without any errors.

After v0.8, an error is thrown. Use something like **SafeMath library** if Solidity version is below v0.8.

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

## 8. Gas DoS

- Looping over dynamic arrays is dangerous. The array may grow so much in future that the operation of the array's deletion/iteration **becomes more than block gas limit making the contract unusable**.
- Hence, avoid such opeartions or **divide the operations into different blocks**. Like empty a part of array and store the indice in storage for next block.

## 9. DAG (C3) Abuse : 

- Read the C3 Linearization in `general notes` if you didn't. If overlooked, C3 Linearization might expose a potential flaw.
- The `super.func()` might yield unexpected results if the linearization decided inheritance order different from what the user expected. See [Here](https://pdaian.com/blog/solidity-anti-patterns-fun-with-inheritance-dag-abuse/).

## 10. Dynamic array/mapping backdoor : 

- [This](https://github.com/Arachnid/uscc/tree/master/submissions-2017/doughoyte) explains it better than anything. 
- TL;DR? Simply, a contract's owner can maliciously manipulate some storage variables if they don't implement strict dynamic array length checks. 

## 11. Writing critical code yourself : 

- **DO NOT EVER WRITE SOMETHING LIKE  `ecrecover` YOURSELF!**. Use industry standard libraries like ones at openzeppelin. 

## 12. Replay Attacks : 

- Often we users to sign messages off chain and then send them on chain later. **This can lead to replay attacks**
- So, to avoid some attacker replaying the sign by **storing the hash of processed transactions**
- And also to avoid replay between different contracts, **include the contract address and chain id too**. 

## 13. Timestamps : 

- There's little error here but worth learning. The `block.timestamp` should not be used to precisely calculate time. It can only be used for large values like days/weeks and above.
- `block.number` to estimate time too is not exactly correct. There might be a change in ethereum's 14 second block time and your hardcoded value might be off a bit too much. Consensys says if you can tolerate a 15 second diff, safe enough to use.
- DO NOT ever ever use `block.timestamp` as source of randomness.

## 14. `msg.sender` vs `tx.origin`: 

- `msg.sender` is the closest entity that triggered the contract. `tx.origin` is the farthest that actually initiated the transaction.
- Using `tx.origin` can be attacked as the legit owner/user can be phished to call a malicious contract and then that contract can gain authorization to the victim contract.
- But `tx.origin` isn't all bad. It can also be used if you know what you're doing. Something like blacklisting an EOA from calling your contract can be done this using it.

## 15. Use the withdrawal pattern over forwarding:

- Often in something like a dividends contract, when someone deposits, the contract accepts the payment and then **tries to send each past depositor their ETH reward**.
- This is bad for multiple reasons. First, the **contract might hit the gas limit**, looking at you  [GovernMental](https://www.reddit.com/r/ethereum/comments/4ghzhv/governmentals_1100_eth_jackpot_payout_is_stuck/).
- Next, what if the depositor is a contract? **This is a huge mess up. Imagine that contract has some code on it's fallback/receive methods**. 
- Now you're thinking that just the gas for an address is increased. **But what if an attacker wantedly creates a contract with `revert` in fallback and deposits?**
- All the other users won't get any dividends and your contract is gone as good if there isn't some owner sweeping method.

## 16. Untrusted Delegatecall : 

- Never ever `delegatecall` to untrusted contracts. This is bad in multitude of ways. 
- The callee contract has power over your storage and even balances. 
- And check out how parity wallet hack happened [here](https://blog.openzeppelin.com/on-the-parity-wallet-multisig-hack-405a8c12e8f7/). This isn't exactly untrusted but overlooked.

## 17. Storage pointers left uninitialized :  

- Check the deep pointer trap section in the `general notes`.

## 18. Reentrancy

- The good ol hack that caused the DAO hard fork. Always use re entrancy guard or use the withdraw pattern instead of forwarding. 
- Use a re entrancy lock from folks at openzeppelin or use the [checks-effects-interactions pattern](https://docs.soliditylang.org/en/latest/security-considerations.html#use-the-checks-effects-interactions-pattern).

## 19. Loose access to `selfdestruct()`

- Don't write `selfdestruct()` code unless you intend it. Make sure that the `selfdestruct` call is made by verified users/multsigs. 
- Check out  [I accidentally killed it](https://www.parity.io/blog/a-postmortem-on-the-parity-multi-sig-library-self-destruct/), Parity wallet again.

## 20. Always check return values : 

- When using `send ` or `call` check return values and make decision.
- Something like `balance -= amountSent` after `send(amountSent)` is a mistake as the send might've failed.

### **21. Expect the worst and have circuit breakers**:

- Don't expect that your code will never be hacked. Always have some pausable functionality
- Decide what to do in worst case and have a multsig contract **pause** your contract if a bug is discovered. Imagine someone discovered a bug and you have to just watch and do nothing while your contract is getting hacked.
- **All the things here are hacks that have happened/documented, there might also be future vulns and having pausability helps**.



## A few things to consider when developing smart contracts : 

1. Testing is more important. Using Remix to quickly prototype or just build something for fun is good. In fact Remix is awesome to learn solidity syntax.
2. But, when you want to build something serious for even for your portfolio or create a Dapp/contract for contributing to a community, **testing is important**.
3. **Always use industry standard libraries like openzeppelin**. And still write tests, their library is tested but your extension code is not.
4. Learn **Waffle testing**. Hardhat users, have a look at the `hardhat-waffle` library [here](https://hardhat.org/plugins/nomiclabs-hardhat-waffle.html). 
5. Use the `solidity-coverage` plugin. Good tests cover most of the code and cases. Use [this plugin](https://www.npmjs.com/package/solidity-coverage).
6. Run **slither** and check if your code has any known vulns. Check out slither [here](https://github.com/crytic/slither). Check out **Mythril** from consensys [here](https://github.com/ConsenSys/mythril) for similar functionality with also traces of transactions. 
7. If you want even more confidence, use a graphing package like [this](https://github.com/fergarrui/ethereum-graph-debugger) or [this](https://github.com/raineorshine/solgraph). They help show you how the control flow is happening in your contract.
8. When you're sure that testing is done and coverage % is good, test on **local fork**. Hardhat can **fork current mainnet state**. And now simulate if everything works good.
9. Then **deploy to testnet**. Developers using `block.<property>` and anything block related should decide which testnet to deploy as many vary from correctly replicating but unstable testnets to fast blocks but not so close testnets.
10. **Most importantly**, always be in the game. Sign up for Week in Ethereum News or immunefi or anything that helps keep you updated. You'll mostly come across any new developements from them. And in the best case, **you find out something yourself** . Also follow @samczsun on twitter and read his blog as he explains many vulns and what he did.





