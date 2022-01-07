# General Notes

##### Types : 

1. `uint8, uint16, uint<X>` are unsigned ints with X bits of capacity. 
2. `int8, int32, int<X>` are signed integers with X bits of capacity.
3. `bool`s are true/false. Default is false.
4. `address` type is a valid address. Comes in 2 flavors, normal `address` and `address payable`. Difference is that payable type has 2 extra methods.  `transfer` and `send`.
5. `Struct` is also a type. But note that it is just a **template**. We need to declare somewhere else potentially inside a mapping or something to **instantiate** the actual variable like, let there be `struct S` template and we need to do something like `S a;` **to create an object a of type S.**
6. Note that **strings and byteX** types are **Big endian.** While other value types are **little endian**. 
7. **Mapping** can have any type as key except user defined types like *Array, Struct and Mapping*. The value can be anything, **even user defined types**
8. Mappings can only be defined in storage, i.e as state variables. Even if something like a **struct contains a mapping**, that struct **can't be instantiated inside a function** as **memory does not allow mappings even if they are inside some allowed type**. 
9. **Enums** are just like C. Can have a value of only one of it's members at a time. Members are indexed from 0. Returning enums returns **uint** as **ABI does not have the concept of Enum**. Also, defualt value of enum is **first member**. 

```solidity
contract Enum {
    // Enum representing shipping status
    enum Status {
        Pending,
        Shipped,
        Accepted,
        Rejected,
        Canceled
    }
    Status public status;
    // Returns uint
    // Pending  - 0
    // Shipped  - 1
    // Accepted - 2
    // Rejected - 3
    // Canceled - 4
    
    // Since enum types are not part of the ABI, the signature of "getChoice"
    // will automatically be changed to "getChoice() returns (uint8)"
    // for all matters external to Solidity.
    function getChoice() public view returns (status) {
        return status;
    }

    // Update status by passing uint into input
    function set(Status _status) public {
        status = _status;
    }
}
```

10. **Struct** is a user defined type. Similar to enums, **structs can also be imported**. Structs are similar to classes. To instantiate a new object, use the constructor. Bear in mind that all new objects are created in memory(as only place we can do this is inside a function) and **if the struct were to contain mapping**, solidity throws an error.
11. Arguments to constructor **must follow parameter order** or specify `{key:value}` object if order isn't followed.
12. **Array** is similar to other langs. `uint[] myArr`is the way to declare dynamic arrays. `uint[10] tenArray`is fixed array. Note that static array values are initialized to **zero** . Note that functions can also return arrays. `return staticOrDynamicArray`is the way. Note that if arrays grow too big, return takes more gas.
13. Note that **memory can have only fixed size arrays** and also **can't use methods like push and pop on memory arrays**.
14. using `delete myArray[i]` **does not shrink the array**. `delete` just changes values to **default value**. 

##### Variable Scopes : 

There are 3 types of variable scopes. 

1. `local`, declared and used inside functions. Destroyed after execution. Not stored on blockchain
2. `state`, declared in the contract scope, stored on blockchain. Accessible by contract functions
3. `global`, accessed by all. Like `msg.sender` and `block.timestamp`.

##### Immutability : 

1. `Constant`. Declared once and can't be changed
2. `Immutable`. Can be changed only on contract instantiation i.e during inside of constructor. 

#### Accounts :

- Contract addresses are derived from deploying user and their nonce.
- Both are treated by EVM as same way called **Accounts**. Every account has a **persistent key value mapping store**. It is called **Storage** and maps **256-bit words to 256-bit words**.

#### Transactions : 

- A transaction is a message from one account to other. Contains **payload** and **ether**. The payload is in **binary** hence the `Abi encoding` and if the target address has code, it is executed based on the payload.
- A transaction to `null` or empty address creates a **contract** with payload as **bytecode**. 
- Out of gas transaction is reverted and **all state changes are discarded**

#### Storage, Memory and Stack : 

- **Storage** as above stated is a mapped key store data. Enumerating storage is not possible. 

- Costly to modify and even more costly to initialise. **Once allocated, storage can't be allocated again, not possible through any function call**. 

- **Memory** is a freshly created space for every message call. Byte level addressable. Reads are **limited to 32 bytes** at once. Writes can be **1 byte or 32 bytes**, (note that this isn't storage and the extra gas for reduced size elements theory(from gas optimisations) doesn't apply here).

- Any reference or attempt to **previously unused memory** slot makes the EVM **Expand it by a word** i.e 32 byte/256 bits. The expansion cost must be **paid in gas** and costs scale **quadratically**. Note that practically **memory** is unlimited. But gas is too much as it increases.

- **EVM is a Stack based machine**. Stack limit is **1024** items with each item being **32 Bytes**. Note that although limited, **stack is the cheapest in terms of gas**.

- Only the topmost **16 Elements** can be accessed at most. That too is only in the case of **COPY (DUP16) or SWAP (SWAP16)**. All other operations take the top 2 only.

- ## Locations : 

- Function arguments are always stored in memory

- State variables are always stored in storage

- Local variables (declared inside function) : 

  1. If **primitves(value types), **stored on **Stack**

  2. If **reference types(arrays, structs)** they default to **storage**, can be explicitly declared as **memory** too. Note that they are simply *references*, so until they are assigned they have no memory/actual storage. Once assigned, behaviour depends. Look below!

     #### *Now for an investigation into the deep pointer trap!*

  Have a look at the following code

  ```solidity
  contract FirstSurprise {
   
   struct Camper {
     bool isHappy;
   }
   
   mapping(uint => Camper) public campers;
   
   function setHappy(uint index) public {
     campers[index].isHappy = true;
   } 
   function surpriseOne(uint index) public {
    Camper c = campers[index]; // Look over here, Line 1
     c.isHappy = false; // Line 2
   }
   
   }
  ```

  1. Now recall that our theory says storage can't be created inside function calls. So what is `Camper c` in `Line 1 `? Our theory so far says they default to **storage**?

  2. Well here's the deal. `Campers c` is actually a **Storage Pointer**. While actual `c` variable is stored inside **memory**, being a storage pointer it points to **storage**!
  3. So the change in `Line 2` changed actual storage as **it is a reference**!
  4. Note that nothing is copied here. **Storage to Memory copy didn't happen**. 

  

  Now take a look at another similar code snippet,

  ```solidity
  contract AnotherSurprise {
   
   struct Camper {
     bool isHappy;
   }
   
   uint public x = 100;
   
   mapping(uint => Camper) public campers;
   
   function setHappy(uint index) public {
     campers[index].isHappy = true;
   }
   
   function surpriseTwo() public {
     Camper storage c; // Line 1, explicitly set to storage
     c.isHappy = false; // Line 2
   }
  ```

  

  1. Previously we didn't explicity say `storage`. That raises a compiler warning of `Variable is declared as storage pointer. Use explicit “storage” keyword to silence this warning.` 
  2. Now we did. Now a new warning pops out. `Uninitialized storage pointer`. 
  3. And surprisingly, **Line 2 makes x value equal to 0!** Why?
  4. Becuase we now explicitly declared as storage, **The pointer by default points to location 0x00 or first slot**. And guess who is chilling out at storage slot 0x00?
  5. That's X. And implicitly what's happening in `Line 2`  is, we are setting value of X to false which is zero.

  *So a wise use case for **storage pointers** is to get read only access if we aren't careful. Can also use for writing provided we know what we're doing with manipulating the state reference.*



### Calls : 

- Errors **bubble up**. If the inner contract we called via message call fails, our contract will throw a **manual error**. Only the gas passed to failed contract is wasted.
- The inner contract or the *callee* in any context will get a fresh instance of memory and the **payload in the call** referred to as **calldata**. 
- To access the payload of that call, **calldata** must be referred. Any return variables will be stored at a location of **caller's memory** preallocated by the caller.
- A minor gotcha, when forwarding gas to a message call, max you can send is 63/64 of your total gas which maxes out to **stack depth of little less than 1000**.
- **Delegate call** is a special call in solidity where code is dynamically loaded from other contract and executed in the state context of calling contract. This makes the **library** feature possible.
- In **create** calls that help contracts create other contracts, compared to craeting from an EOA the only diff is that the address of new contract is available on **top of stack** of creator contract.



### Self Destruct : 

- Contract can call `selfdestruct` on itself and then the code and state is removed from the blockchain. 
- Note that using `selfdestruct` is not a totally deleting the contract. I think that nodes syncing from ground up can stop before the block `selfdestruct` was called and see it's state and code. (Think? Need some more info)
- Even if contract's code doesn't have `selfdestruct` function, it can still be done via the `delegatecall` or `callcode`. (Note that `callcode` was an older depreciated version of `delegatecall` where `msg.sender` was the latest caller, `callcode` caller instead of `callcode` caller's caller i.e the `user` ).



1. Solidity does not support default exports like JS. 
2. Importing in this way, `import "filename" ` is a bad choice as namespace will be polluted. Can be used with **standard third party files** like OpenZeppelin/Uniswap
3. `import "filename" as symbolName` lets us access it's contents like `symbolName.variable` which is better.



### Storage vs Memory vs Calldata : 

1. **Storage is persistent till network lives. Memory is persistent till function call completes. Calldata is persistent till network lives, but is not modifyable and is not tied to the contract.** More on that later..
2. **Storage** is the actual data taking space when ETH client is downloaded. So it is a disk write. So it costs more gas
3. **Memory** is data during execution. It can only be created inside functions as local variables. Miner's RAM is where memory data lives. After each function call completes it is wiped. Costs less gas than storage
4. **Calldata** is the input data **generated by the user and sent as a transaction**. All the parameters the user intended to give the contract **are located in the calldata**. It's the **data** field in a transaction.

##### Why does it matter to the contract/EVM?

1. It gives the flexibility to choose where we store data. Memory is cheaper and can be used to read and write intermediate computations.
2. Functions can also be typed and can be told to find which data from where

##### Copies and References : 

1. Note that **calldata** can only be read. It is part of a **signed ethereum transaction**, you can't change it. So `uint calldata imaginaryNum = 6` does not make sense.
2. Assigning between storage and memory **creates copies**.
3. Assigning between **memory to memory** or **storage to storage** creates **References**! Be careful with the reference traps. *Refer to above code for traps*

##### Here comes the stack :

1. EVM is a stack based machine. Sure permanent storage and memory exist but it can't just be working with them alone. Computers are **register based machines**, they store intermediate values in these registers and use them to compute further.
2. **EVM instead of registers uses stack.** 
3. How does it matter to contract? Note that **value types** like uint, bool when declared inside a function **are created on the stack**!
4. Similar to memory, **stack is newly created when a function is called. Hence the word stacktrace**.
5. *Security Note* : EVM allows an opcode that let's you swap last 16th value. If a function invokes other function, the stack might propagate and the inner function may access the caller's variables. (*Self idea, needs reference to make sure*).



## Functions : 

1. Functions in solidity can take in and also return multiple values.
2. Destructuring ` (a, b, c) = returnsThree()` is also present in solidity. But note that brackets aren't squared like in JS
3. Functions **cannot use maps for inputs or outputs**
4. If a function returns something, needs to be specified in it's signature. like `function myFunc returns (uint)`. For functions returning nothing, **don't specify the** `returns` keyword and type, something like `returns (void)`  isn't valid in solidity.
5. `return` statement can be **omitted** if **name is specified after returns keyword**. Below code is perfectly returns though there's no `return` statement. 

```solidity
function assigned()
        public
        pure
        returns (
            uint x,
            bool b,
            uint y
        )
    {
        x = 1;
        b = true;
        y = 2;
    }
```

6. `View` functions mean that they **only read. No state change**. `Pure` functions mean they **neither read nor write from state**. Note that the constraints apply only on state. You can still create memory objects and interact with the user's calldata in a normal way.

### Visibility : 

1. `Public` : Can be called internally(from inside current contract) and also externally(other contracts or EOAs)
2. `private` : Can only be called from inside of a the contract internally.
3. `internal` : Only internally from inside a contract or by children inheriting it. Something on the lines of Java's protected.
4. `external` : Can only be called externally. To call from inside the contract, use `this.f()` 

### About return values : 

1. Calling functions as an EOA/User : 
   - **Only constant i.e view or pure functions can return values to users**.
   - **Non constant i.e state changing functions return a TXN HASH and cost gas**
   - The only way to extract any values from state changing function to user/UI is **Listening to events**.
2. Calling functions of a contract from another contract : 
   - Both constant and non constant functions can return values when the caller is a contract. 
   - Note that **constant functions too cost a little when called from a contract** . 
   -  ---------NEED TO ADD SOME HERE--------

### `require`, `revert` and `assert`:

1. There are two kinds of **Errors** that EVM can throw. `Error` and `Panic`. 
2. Consider `Error`  like a soft check exception you can throw in your contracts. Like **Input validation or return values or state checks**
3. `Panic` is the actual compiler/EVM screaming that something terribly bad has happened. Likes of them are *divide by zero, overflows, negative indices, large memory allocation*. There's also a special one called *compiler inserted panic*. More on that later.
4. `Error` is the exception thrown **by your code**. Your code intended to throw that error and is not some fatal error like `panic` thrown by EVM.

### Ways you can throw Error :  

1. Now, `require` syntax is `require(checkReturnsTrueOrFalse, Error)`. The `checkReturnsTrueOrFalse` should evaluate to a `bool`.
2. If it evaluates to a `false`, an **Error** is thrown with `Error` as description.
3. Syntax for revert is `revert(descriptionString)` or just `revert()`. This also throws an error.

##### When to use require vs revert : 

- `require` can only perform small checks and then throw error.

- If the error checking code is complex then use `revert` like this,

  ```solidity
  if(complexErrorCheck == false){
  	revert("Error Description");
  }
  ```

- Note that `require` only creates and throws `Error` of type `Error("Description String")`. **Custom Error objects can't be used**, use **revert for that**

- `revert` can throw custom errors like this,

```solidity
if(complexErrorCheck == false){
	revert customError();
}
```

### Ways a Panic can occur : 

1. Remeber **compiler inserted panic** from above, let's talk about that.
2. Along with examples like above, panic can occur in two extra **compiler/user invoked ways**
   - `Panic 0x00` is thrown by **compiler inserted panic** 
   - `Panic 0x01` is thrown by **Failing an assert condition**!

##### Assert : 

1. Assert syntax is `assert(onFalseThrowPanic)`. If the check becomes false, a panic `Panic 0x01` is thrown.
2. What's strange is that internally, **assert actually uses revert to throw a customError**. Is that customError just called `Panic`? (*self idea, need references*).



## Modifiers : 

1. Function modifiers are special code that **can be injected before a function on which the modifier is applied is called**,.
2. They can be used for things like **Access control, some input checks** and most importantly **Re-Entrancy Guards**.
3. Actually, modifiers are invoked **before executing the function**. But, the modifier can choose **Where to pass the power back to it's function**. The `_;` is used to **execute the actual function**. Check below

```solidity
contract Mutex {
    bool locked;
    modifier noReentrancy() {
        require(
            !locked,
            "Reentrant call."
        );
        locked = true;
        _;      // Look over here. Notice the _;
        locked = false;
    }

    /// This function is protected by a mutex, which means that
    /// reentrant calls from within `msg.sender.call` cannot call `f` again.
    /// The `return 7` statement assigns 7 to the return value but still
    /// executes the statement `locked = false` in the modifier.
    function f() public noReentrancy returns (uint) {
        (bool success,) = msg.sender.call("");
        require(success);
        return 7;
    }
}
```

4. In the above code when `f` function is called, first `noReentracy` modifier is executed. While execution, the modifier performs it's logic and then calls it's function using `_;`. The `_;` is just a placeholder for modifier to pass the power to it's function. **Note that additional logic can also be performed after the function executes**.
5. The modifier above also does some state changes **after function has executed**.
6. So modifiers seem to be a **wrapper function around actual functions**. Hence also note that **arguments to function are also forwarded to the modifiers**. 
7. But arguments need to be passed to modifiers in the function signature like `function someFunction(address _user) notSpecificAddress("0x......")`, here the value is hardcoded but it can be a **state variable or an argument passed down by the function**.
8. Note that the placement of `_;` is key to writing secure code. So for starters, include it at end of custom modifers.

### Events : 

1. Events are inheritable members of contracts. The syntax is 

```solidity
event Deposit(
        address indexed _from,
        bytes32 indexed _id,
        uint _value
    );
```

2. And to emit them, 

`emit Deposit(msg.sender, _id, msg.value);`

1. Any web3 JS library can then subscibe to these events. 

## Purpose of Events?

- Note that **only constant i.e view or pure functions can return values**. 
- So how do you **update your Dapp UI if write/state changing functions are called?** That's where events step in!



#### Constructors : 

1. Constructors of a contract **run before deployment of the contract**. The contract deployment cost rises **linearly with size of the code**. But note that **constructor code and internal functions code used only in constructor is not billed in gas **

2. State variables are **initialized to default values** even before running the constructor.

3. A contract needs to implement all it's parent contract's constructor compulsorily. Else it is declared **abstract**. 

4. 2 ways to do that, 

   - Specify in the inheritance list itself. **Better suited when args are constants**

   - ```solidity
     // SPDX-License-Identifier: GPL-3.0
     pragma solidity >=0.7.0 <0.9.0;
     
     contract Base {
         uint x;
         constructor(uint _x) { x = _x; }
     }
     
     // Either directly specify in the inheritance list...
     contract Derived1 is Base(7) {
         constructor() {}
     }
     ```

   - Specify in the modifiers of constructor. **Used when args are actually args to child constructors**

     ```solidity
     // SPDX-License-Identifier: GPL-3.0
     pragma solidity >=0.7.0 <0.9.0;
     
     contract Base {
         uint x;
         constructor(uint _x) { x = _x; }
     }
     
     
     contract Derived2 is Base {
         constructor(uint _y) Base(_y * _y) {}
     }
     ```

5. **Inheritance is linearized**. Which means always specify the inheritance directive `contract child is grandParent, Parent` like that. Doing something like `contract child is Parent, grandParent` does not work. Simply put, **List highest parents first in inheritance tree. TOP to BOTTOM becomes **

   **`contract child is Top1, Top2, Top3, oneLevelAboveChild `** .

6. Note **Parent constructors are run not in modifier specified way. They always run in the same linearized manner**. See below

   ```solidity
   contract Base1 {
       constructor() {}
   }
   
   contract Base2 {
       constructor() {}
   }
   
   // Constructors are executed in the following order:
   //  1 - Base1
   //  2 - Base2
   //  3 - Derived1
   contract Derived1 is Base1, Base2 {
       constructor() Base1() Base2() {}
   }
   
   // Constructors are executed in the following order:
   //  1 - Base2
   //  2 - Base1
   //  3 - Derived2
   contract Derived2 is Base2, Base1 {
       constructor() Base2() Base1() {} // Modifier order does not 																				// matter. Follows linearised 																			//order
   }
   
   // Constructors are still executed in the following order:
   //  1 - Base2
   //  2 - Base1
   //  3 - Derived3
   contract Derived3 is Base2, Base1 {
       constructor() Base1() Base2() {} // Same here
   }
   ```

7. Becuase `Base1` and `Base2` don't have any inheritance relation, the linearization order is **the order we have given in the `is <>` directive **. Hence constructors also run in that order i.e **linearised order**. Had they been related in a inheritance relation, **we need to make sure out `is <>` directive is linearised** and constructors run in that order.

#### Inheritance : 

1. If a contract inherits from multiple contracts and both those contracts inherit from same base contract (**google diamond pattern**), then **only one copy of top most contract is created**. See below. Similar to **Virtual Inheritance** from C++.

```solidity
contract Owned {
	string name = "John"
}

contract Destructible is Owned {
	// string name = "Something"; not allowed as it shadows
	constructor(){
		name = "Joseph" // This directly points to parent's name
	}
}

contract Named is Owned, Destructible { // Inherits only a single 'Owned', not twice.
   
}

```

### A case analysis : 

1. A contract has defined some functions as `virtual`. Another contract inherits from it.

   - This is the case with contracts like `ERC20`. They define some functionality and let children **extend** that functionality. Note that the extension is a **Choice**.
   - For example, the `transfer` function of `ERC20` is virtual. Which means you can implement your own `transfer` function using override, **while also using openzeppelin's code by just saying** `super.transfer()`. 

2. A contract has **declared some functions as virtual** but did **not define them**. 

   - This makes the above contract `abstract`. Any child inheriting them **can't be instantiated without defining that functionality**.
   - Contrast to above, this isn't a choice, it's a **burden** on the child contract.
   - But why? Consider a case where there is an `interface` (more on **interface later**) or `abstract contract` called `willDoSomething`. 
   - This tells the compiler/guarantees the user that **contracts inheriting `willDoSomething` will actually do something**.
   - This can be used to make sure that **A standard is adhered and implemented**. Hence the word `IERC20` in libraries. `IERC20` is the interface telling us the rules of the **ERC20 standard**. The interface itself `IERC20` **does not ** implement anything. It **Forces** inheriting members to complete the functionality so that standards are met.
   - A little gotcha, `abstract contract`s cannot cannot override an implemented virtual function with an unimplemented one. In simple words, **you can enforce your children to write/define an implementation, but can't deny them the functionality passed from above**. 

3. An `abstract contract` or `interface` is defined/imported and not inherited. Used as a **variable inside a contract**

   - This is the case when you **want to assure compiler that it need not throw undefined** errors.
   - Consider a case where your contract wanted to **use Uniswap** contracts inside your contract. How do you tell your compiler that **Uniswap docs guarantee this functionality, so I'm gonna use it**?
   - This is where interface comes in. See below : 

   ```solidity
   import '@uniswap/v3-periphery/contracts/interfaces/ISwapRouter.sol';
   
   contract swapExample {
     ISwapRouter public immutable swapRouter;
   
     constructor(ISwapRouter _swapRouter) {
             swapRouter = _swapRouter;
         }
         
         
     function swapExactInputSingle(uint256 amountIn) external returns (uint256 amountOut) 
     {
       ISwapRouter.ExactInputSingleParams memory params = // Using interface's struct
       ISwapRouter.ExactInputSingleParams({
       tokenIn: DAI,
       tokenOut: WETH9,
       fee: poolFee,
       recipient: msg.sender,
       deadline: block.timestamp,
       amountIn: amountIn,
       amountOutMinimum: 0,
       sqrtPriceLimitX96: 0
       });
   
       // The call to `exactInputSingle` executes the swap.
       amountOut = swapRouter.exactInputSingle(params); // Using interface's function
      }
   }
   
   ```

   - Uniswap has developed and deployed their **SwapRouter** contract somewhere else but your compiler needs to **have a reference** to check what functions and structs/vars it supports or not.
   - For that purpose, uniswap has released an **Interface** so that you can import it and **make the compiler not yell while also getting typed reference**.

#### `Interface` vs `abstract contract` : 

1. Both are used to force reference implementation of children as told above.
2. Differences : 
   - `abstract contract` is still a contract. It can **have state variables, inherit from other contracts, have a contructor and even define some modifiers ** . All that a normal contract can do, just instantiation can't be done.
   - `interface` cannot do all that. `interface` purely is a **template** forcing inheriting children to implement it. So just a template, nothing like a contract. So it **cannot have state, cannot define ANY function implemented, cannot have a contructor, cannot declare modifiers** and **all functions must be virtual and external** although **virtual keyword** is implicit.
   - But the template extenstion is still possible. `abstract contract`s can inherit whatever they want and `interfaces` can only inherit other interfaces.

#### Function Overriding : 

1. Functions in base contracts need to specify that they can be overriden by child functions using the `virtual` modifier.
2. Functions in child contracts need to specify which functions they are overriding by using the **same signature** as base contract function. I.e **same name and return type**, and also need to specify the `override` modifier.
3. When a child function calls `super.foo()`, solidity searches for it **in parents one level above and in the order** **right to left of `is` directive**(if multiple level 1 parents implement same function).
4. You can also always explicitly call a function of a contract in the inheritance path by specifying which contract you are referring to `twoLevelsUpParent.foo()`.
5. `Override` needs to specify which contract's functions you are overriding when multiple inheritance is in play. See below

```solidity
contract A {
    uint public x;
    function setValue(uint _x) public virtual {
        x = _x;
    }
}

contract B {
    uint public y;
    function setValue(uint _y) public virtual {
        y = _y;
    }
}

contract C is A, B {
    function setValue(uint _x) public override(A,B) {
        A.setValue(_x);
    }
}
```

5. Functions can be both **virtual and override** at the same time. So they can implement/extend some functionality via `override` and also force children to extend this functionality through `virtual`.
6. Note that you can't override parent's `external` function and then try something like `super.this.f()`. Simply put, you can't call parent's external functions if you've already overriden them in the child.