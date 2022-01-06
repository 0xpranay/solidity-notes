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
7. But arguments need to be passed to modifiers in the function signature like `function someFunction(address _user) notSpecificAddress("0x......")`, here the value is hardcoded but it can be a **state variable, argument passed down by the function**.
8. 