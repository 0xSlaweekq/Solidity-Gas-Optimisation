# Solidity Gas Optimization

Solidity Gas Optimization Tips taken from various sources (sources listed in the #credits section below). Make pull requests to contribute gas alpha.

## 1- Change orders of storage slot (20 000 gas saved at deployment)

Ethereum storage is composed of slots of 32 bytes, the problem is that writing is expensive. (Up to 20000gas using “cold” writing)

**Let’s say you have 3 variables:**

```solidity
uint128 a; //Slot 0 (16 bytes)
uint256 b; //Slot 1 (32 bytes)
uint128 c; //Slot 2 (16 bytes)
```

They uses 3 different storage slots

```solidity
uint256 b; //Slot 0 (32 bytes)
uint128 a; //Slot 1 (16 bytes)
uint128 c; //Slot 1 (16 bytes)
```

### Packing Structs

A common gas optimization is “packing structs” or “packing storage slots”. This is the action of using smaller types like uint128 and uint96 next to each other in contract storage. When values are read or written in contract storage a full 256 bits are read or written. So if you can pack multiple variables within one 256 bit storage slot then you are cutting the cost to read or write those storage variables in half or more.

```solidity
// Unoptimized
struct MyStruct {
  uint256 myTime;
  address myAddress;
}

//Optimized
struct MyStruct {
  uint96 myTime;
  address myAddress;
}
```

In the above a `myTime` and `myAddress` state variables take up `256` bits so both values can be read or written in a single state read or write.
But if I align the 3 slots well, I can economize 1 slot. (a and c variable will be in the same slot)

Therefore the is 1 less used slots at the deployment. (So 20 000 gas saved)

**Code Example:**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity >=0.8.7;

contract withoutPacking {
    struct MyStruct {
        uint256 a; // 32 bytes
        address b; // 20 bytes
        uint256 c; // 32 bytes
    }
    MyStruct myStruct;
    // Function Execution Cost = 87614
    function updateStruct() public {
        myStruct = MyStruct({a: 5, b: msg.sender, c: 6});
    }
}

contract withPacking {
    struct MyStruct {
        uint256 a; // 32 bytes
        address b; // 20 bytes
        bool c;    // 1 bytes
    }
    MyStruct myStruct;
    // Function Execution Cost = 65547
    function updateStruct() public {
        myStruct = MyStruct({a: 5, b: msg.sender, c: true});
    }
}
```

As you can see, we have considerably lower gas than the previous one and the gas saved by packing is 87614 — 65547= 22067, which is a very good saving. This drastic change in gas is due to the cold and warm access concept which is discussed above. Even if we pack more variables it will still cost low than the first one as you can see below.

**Code Example:**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity >=0.8.7;

contract withPackingMore {
    struct MyStruct {
        uint256 a;  // 32 bytes
        address b;  // 20 bytes
        bool c;     // 1 bytes
        uint40 d;   // 5 bytes
        bool e;     // 1 bytes
        uint32 f;   // 3 bytes
    }
    MyStruct myStruct;
    // Function Execution Cost = 65628
    function updateStruct() public {
        myStruct = MyStruct({a: 5, b: msg.sender, c: true, d: 3, e: true, f: 23});
    }
}
```

```solidity
bool // 1 bytes
address // 20 bytes
uint8 // 1 bytes
uint16 // 2 bytes
uint24 // 3 bytes
uint32 // 4 bytes
uint40 // 5 bytes
uint64 // 8 bytes
uint96 // 12 bytes
uint128 // 16 bytes
uint256 // 32 bytes
```

## 2- Caching the length in for loops

Reading array length at each iteration of the loop takes 6 gas (three for mload and three to place memory_offset ) in the stack. Caching the array length in the stack saves around 3 gas per iteration. I suggest storing the array’s length in a variable before the for-loop.

**Example of an array arr and the following loop:**

```solidity
for (uint i = 0; i < length; i++) {
    // do something that doesn't change the value of i
}
```

In the above case, the solidity compiler will always read the length of the array during each iteration.

If it is a storage array, this is an extra sload operation (100 additional extra gas (EIP-2929) for each iteration except for the first),
If it is a memory array, this is an extra mload operation (3 additional gas for each iteration except for the first),
If it is a calldata array, this is an extra calldataload operation (3 additional gas for each iteration except for the first).

**These extra costs can be avoided by caching the array length (in the stack):**

```solidity
uint length = arr.length;
for (uint i = 0; i < length; i++) {
    // do something that doesn't change arr.length
}
```

In the above example, the sload or mload or calldataload operation is only called once and subsequently replaced by a cheap dupN instruction. Even though mload, calldataload and dupN have the same gas cost, mload and calldataload needs an additional dupN to put the offset in the stack, i.e., an extra 3 gas.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity >=0.8.7;

contract withoutLengthCaching {
    uint256[] private _arr = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10];
    // Function Execution Cost = 26627
    function getVal() public view returns (uint256) {
        uint256 sum = 0;
        for (uint i = 0; i < _arr.length; i++) {
            sum += _arr[i];
        }
        return sum;
    }
}

contract withLengthCaching {
    uint256[] private _arr = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10];
    // Function Execution Cost = 26610
    function getVal() public view returns (uint256) {
        uint256[] memory arr = _arr;
        uint length = arr.length;
        uint256 sum = 0;
        for (uint i = 0; i < length; i++) {
            sum += arr[i];
        }
        return sum;
    }
}
```

## 3- The increment in the for loop’s post condition can be made unchecked

`unchecked { ++i;}` is cheaper than `i++;` & `i=i+1;`

In Solidity 0.8+, there’s a default overflow check on unsigned integers. It’s possible to uncheck this in for-loops and save some gas at each iteration, but at the cost of some code readability, as this uncheck cannot be made inline.

**To sum up, the best gas-optimized loop will be:**

```solidity
uint length = arr.length;
for (uint i; i < length;) {
  unchecked {
    ++i;
  }
}
```

## 4- Prefer Mapping Instead of Array:

Mapping in Solidity costs less gas than the arrays. If you don’t have strict requirements prefer to use mappings.

**Consider the following code:**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity >=0.8.7;

contract UsingArray {
  uint256[] myArr = [1];
  // gas cost = 229,417
  function setValues() external {
    for(uint i; i < 10; ) {
      myArr.push(i);

      unchecked {
        ++i;
      }
    }
  }
}

contract UsingMapping {
  mapping(uint256 => uint256) myMap;
  // gas cost = 222,538
  function setValues() external {
    for(uint i; i < 10; ) {
      myMap[i] = i;

      unchecked {
        ++i;
      }
    }
  }
}
```

## 5- Use Modifiers Instead of Functions To Save Gas

**Example of two contracts with modifiers and internal view function:**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.9;
contract Inlined {
    function isNotExpired(bool _true) internal view {
        require(_true == true, "Exchange: EXPIRED");
    }
function foo(bool _test) public returns(uint){
            isNotExpired(_test);
            return 1;
    }
}
// SPDX-License-Identifier: MIT
pragma solidity 0.8.9;
contract Modifier {
modifier isNotExpired(bool _true) {
        require(_true == true, "Exchange: EXPIRED");
        _;
    }
function foo(bool _test) public isNotExpired(_test)returns(uint){
        return 1;
    }
}
```

**Differences:**

```js
Deploy Modifier.sol
108727
Deploy Inlined.sol
110473
Modifier.foo
21532
Inlined.foo
21556
```

This with 0.8.9 compiler and optimization enabled. As you can see it's cheaper to deploy with a modifier, and it will save you about 30 gas. But sometimes modifiers increase code size of the contract.

## 6- Use Internal View Functions in Modifiers To Save Bytecode

It is recommended to move the modifiers require statements into an internal virtual function. This reduces the size of compiled contracts that use the modifiers. Putting the require in an internal function decreases contract size when a modifier is used multiple times. There is no difference in deployment gas cost with private and internal functions.

**With Solidity 0.8.14 and optimisations on (200):**

```solidity
contract Ownable is Context {
    address public owner = _msgSender();
modifier onlyOwner() {
        require(owner == _msgSender(), "Ownable: caller is not the owner");
        _;
    }
}
contract Ownable2 is Context {
    address public owner = _msgSender();
modifier onlyOwner() {
        _checkOwner();
        _;
    }
function _checkOwner() internal view virtual {
        require(owner == _msgSender(), "Ownable: caller is not the owner");
    }
}
// This is deployment gas cost for each function
// 0: 107172
// 1: 145772
// 2: 181610
// 3: 198170
// 4: 214532
// 5: 241059
contract T1 is Ownable {
    event Call(bytes4 selector);
    function f0() external onlyOwner() { emit Call(this.f0.selector); }
    function f1() external onlyOwner() { emit Call(this.f1.selector); }
    function f2() external onlyOwner() { emit Call(this.f2.selector); }
    function f3() external onlyOwner() { emit Call(this.f3.selector); }
    function f4() external onlyOwner() { emit Call(this.f4.selector); }
}
// 0: 107172
// 1: 147908
// 2: 165818
// 3: 183506
// 4: 192500
// 5: 211682
contract T2 is Ownable2 {
    event Call(bytes4 selector);
    function f0() external onlyOwner() { emit Call(this.f0.selector); }
    function f1() external onlyOwner() { emit Call(this.f1.selector); }
    function f2() external onlyOwner() { emit Call(this.f2.selector); }
    function f3() external onlyOwner() { emit Call(this.f3.selector); }
    function f4() external onlyOwner() { emit Call(this.f4.selector); }
}
```

## 7- >= is cheaper than >

Non-strict inequalities (>=) are cheaper than strict ones (>). This is due to some supplementary checks (ISZERO, 3 gas)).

```solidity
uint256 public gas;
function checkStrict() external {
        gas = gasleft();
        require(999999999999999999 > 1); // gas 5017
        gas -= gasleft();
    }
function checkNonStrict() external {
        gas = gasleft();
        require(999999999999999999 >= 1); // gas 5006
        gas -= gasleft();
    }
```

## 8- > 0 is cheaper than != 0 sometimes

!= 0 costs less gas compared to > 0 for unsigned integers in require statements with the optimizer enabled. But > 0 is cheaper than !=, with the optimizer enabled and outside a require statement. https://twitter.com/gzeon/status/1485428085885640706

**Example with optimizer disabled:**

```solidity
uint256 public gas;
function check1() external {
        gas = gasleft();
        require(99999999999999 != 0); // gas 22136 --disabled optimizer
        gas -= gasleft();
    }
function check2() external {
        gas = gasleft();
        require(99999999999999 > 0); // gas 22136 --disabled optimizer
        gas -= gasleft();
    }
function check3() external {
        gas = gasleft();
        if (99999999999999 != 0){ // 22149 gas --disabled optimizer
            uint256 i = 123;
        }
        gas -= gasleft();
    }
function check4() external {
        gas = gasleft();
        if (99999999999999 > 0){ // 22152 gas --disabled optimizer
            uint256 i = 123;
        }
        gas -= gasleft();
    }
```

**Example with optimizer enabled:**

```solidity
uint256 public gas;
function check1() external {
        gas = gasleft();
        require(99999999999999 != 0); // gas 22106 --enabled optimizer
        gas -= gasleft();
    }
function check2() external {
        gas = gasleft();
        require(99999999999999 > 0); // gas 22117 --enabled optimizer
        gas -= gasleft();
    }
function check3() external {
        gas = gasleft();
        if (99999999999999 != 0){ // 22106 gas --enabled optimizer
            uint256 i = 123;
        }
        gas -= gasleft();
    }
function check4() external {
        gas = gasleft();
        if (99999999999999 > 0){ // 22105 gas --enabled optimizer
            uint256 i = 123;
        }
        gas -= gasleft();
    }

```

Gas savings: it will save you about 10 gas.

```
To sum up on 0.8.15:
  Without optimizer:
    In require:
        `> 0` equals to `!= 0`
    Outside require:
        `> 0` more expensive than `!= 0`
  With optimizer:
    In require:
        `> 0` more expensive than `!= 0`
    Outside require:
        `> 0` cheaper than `!= 0`
```

## 9- Caching Storage Variables in Memory To Save Gas

Anytime you are reading from storage more than once, it is cheaper in gas cost to cache the variable in memory: a SLOAD cost 100gas, while MLOAD and MSTORE cost 3 gas.

**Gas savings: at least 97 gas.**

```solidity
// struct LockPosition use 3 slots
// struct LockPosition {
// address owner;
// uint256 unlockAt;
// uint256 lockAmount;
//}
function unlock(uint256 _nftIndex) external nonReentrant {
         LockPosition memory position = positions[_nftIndex]; // gas: costing 3 SLOADs while only lockAmount is needed twice.
         //Replace "memory" with "storage" and cache only position.lockAmount
require(position.owner == msg.sender, "unauthorized");
         require(position.unlockAt <= block.timestamp, "locked");
delete positions[_nftIndex];
jpeg.safeTransfer(msg.sender, position.lockAmount);
emit Unlock(msg.sender, _nftIndex, position.lockAmount);
     }
```

## 10- Unchecked Blocks:

Before Solidity 0.8.0, arithmetic operations would always wrap in case of underflow or overflow leading to the widespread use of libraries that introduce additional checks. Since Solidity 0.8.0, all arithmetic operations revert on overflow and underflow by default, but this behavior also includes some checks that cause the gas.

An unchecked block was introduced in Solidity that bypasses certain safety checks performed by the Solidity compiler. It is typically used when the developer wants to optimize the gas usage of their smart contract. However, using unchecked blocks also carries risks and by bypassing certain safety checks, the developer is taking on additional responsibility to ensure the code is secure and not vulnerable to attacks.

**The following functions are executed at the optimization of 200 runs:**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity >=0.8.16;

contract WithoutUnchecked {
uint256 private \_value = 0;
// function execution cost = 22322 gas
function increment() public {
\_value += 1;
}
}

contract WithUnchecked {
uint256 private \_value = 0;
// function execution cost = 22225 gas
function increment() public {
unchecked {
\_value += 1;
}
}
}
```

As you can see using the unchecked block saved us 22322–22225 = 97 gas but only use this unchecked block if you are sure that the value will not overflow or underflow.

## 11- Storage — Cold Access vs Warm Access:

Cold access typically refers to accessing data that is stored on the blockchain but is not currently in the cache or memory of the executing contract while warm access typically refers to accessing data that is already in the cache or memory of the executing contract. Warm Access is a faster process and as a result, it requires less gas than cold access which can be seen in the following image taken from Ethereum Yellow Paper:

Cold and Warm Access Cost — Ethereum Yellow Paper
Cold and Warm Access Cost — Ethereum Yellow Paper
We can save the gas by caching (warming) the value, and this makes a great difference if you are using a variable multiple times or looping through an array.

**Checkout the following that is compiled at the optimization of 200 runs:**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity >=0.8.16;

contract WithoutCaching {
    uint256[] private _arr = [1, 2, 3, 4, 5, 6, 7, 8, 9];
    // function execution cost = 25506 gas
    function getVal() public view returns (uint256) {
        uint256 sum;
        for (uint i = 0; i < _arr.length; i++) {
            sum += _arr[i];
        }
        return sum;
    }
}

contract WithCaching {
    uint256[] private _arr = [1, 2, 3, 4, 5, 6, 7, 8, 9];
    // function execution cost = 24228 gas
    function getVal() public view returns (uint256) {
        uint256 sum;
        uint256[] memory arr = _arr; // caching the array
        for (uint i = 0; i < arr.length; i++) {
            sum += arr[i];
        }
        return sum;
    }
}
```

## 12- Uint256 vs Other Datatypes:

In solidity either we store in boolean, uint128, uint256, or any other datatype, under the hood, it takes 32 bytes. So when we use a datatype other than uint256, the gas cost increases because either datatype requires more opcodes for offsetting purposes.

**This can be seen in the following code:**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity >=0.8.7;

contract withUint256 {
uint256 a = 1;
// 2246
function getUint() public view returns (uint256) {
return a;
}
}

contract withUint128 {
uint128 a = 1;
// 2258
function getUint() public view returns (uint128) {
return a;
}
}

contract withBoolean {
bool a = true;
// 2258
function getBool() public view returns (bool) {
return a;
}
}
```

## 13- Calldata vs Memory:

In older versions of solidity, the arguments were calldata by default in external functions while arguments were memory by default in public functions dues to which external functions used to cost less. Now we can define these arguments explicitly.

Calldata costs less gas than the memory because calldata parameters are not modifiable that’s why we don’t need to create a new memory variable unnecessarily.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity >=0.8.7;

contract withMemory {
    // Function Execution Cost = 22011
    function doNothing(bytes memory _bytes) external {}
}

contract withCalldata {
    // Function Execution Cost = 21865
    function doNothing(bytes calldata _bytes) external {}
}
```

## 14- Memory Explosion:

Memory is very cheap to allocate as long as it is small but past a certain point i.e., 32 kilobytes of memory storage in a single transaction, the memory cost enters into a quadratic section and the formula for this calculation is given in Ethereum Yellow Paper as follows.
So, you can see the implication of creating a very large array in the following code:

As you can see calldata saved us **22011–21865 = 146 gas.**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity >=0.8.7;

contract smallArraySize {
    // Function Execution Cost = 21,903
    function checkArray() external {
        uint256[100] memory myArr;
    }
}

contract LargeArraySize {
    // Function Execution Cost = 276,750
    function checkArray() external {
        uint256[10000] memory myArr;
    }
}

contract VeryLargeArraySize {
    // Function Execution Cost = 20,154,094
    function checkArray() external {
        uint256[100000] memory myArr;
    }
}
```

**So, if we subtract the contract execution cost i.e., 21,000 then the cost for each function becomes:**

1. 21,903–21,000 = 903 gas
2. 276,750–21,000 = 255,750 gas
3. 20,154,094–21,000 = 20,133,094 gas

Now, these calculations clearly show how gas cost increased quadratically with the size of the array.

## 15- Revert Early:

We should revert early because reverting early saves the gas which is then returned to the caller.

**The following code is a good example of reverting as early as possible:**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity >=0.8.7;

contract RevertEarly {
    uint256 private _x;
    // In case of revert, Function Execution Cost = 21,527
    function revertTest(uint256 x) external {
        require(x < 5, "x must be less than 5");
        _x = x;
    }
}

contract RevertLate {
    uint256 private _x;
    // In case of revert, Function Execution Cost = 43,636
    function revertTest(uint256 x) external {
        _x = x;
        require(x < 5, "x must be less than 5");
    }
}
```

## 16- Indexed Events:

Using the indexed keyword for value types like uint, bool, and address can reduce gas costs but this optimization only applies to value types, as indexing bytes and strings can increase gas costs compared to their unindexed versions.

**Consider the following code:**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity >=0.8.7;

contract UnindexedEvents {
    event Bid(uint256 tokens, address bidder, uint256 when);
    // cost = 23,544
    function bid(uint256 tokens) public {
        emit Bid(tokens, msg.sender, block.timestamp);
    }
}

contract IndexedEvents {
    event Bid(
        uint256 indexed tokens,
        address indexed bidder,
        uint256 indexed when
    );
    // cost = 23,503 gas
    function bid(uint256 tokens) public {
        emit Bid(tokens, msg.sender, block.timestamp);
    }
}
```

## 17- Use of Bit shift operators

Check if arithmethic operations can be achieved using bitwise operators, if yes, implement same operations using bitwise operators and compare the gas consumed. Usually bitwise logic will be cheaper (ofc we cannot use bitwise everywhere, there some limitations)

**Note:** Bit shift `<<` and `>>` operators are not among the arithmetic ones, and thus don’t revert on overflow.

**Code example:**

```diff
-    uint256 alpha = someVar / 256;
-    uint256 beta = someVar % 256;
+    uint256 alpha = someVar >> 8;
+    uint256 beta = someVar & 0xff;


-    uint 256 alpha = delta / 2;
-    uint 256 beta = delta / 4;
-    uint 256 gamma = delta * 8;
+    uint 256 alpha = delta >> 1;
+    uint 256 beta = delta >> 2;
+    uint 256 gamma = delta << 3;

```

## 18- Public vs External (External is cheaper)

The `public` visiblilty modifier is equivalent to `external` plus `internal`. In other words, both `public` and `external` can be called from outside your contract (like MetaMask), but of these two, only `public` can be called from other functions inside your contract.

Because `public` grants more access than `external` (and is costlier than the latter), the general best practice is to prefer `external`. Then you can consider switching to `public` if you fully understand the security and design implications.

**Code Example:**

```solidity
pragma solidity 0.8.10;

contract Test {

    string message = "Hello World";

    // Execution cost: 24527
    function test() public view returns (string memory){
         return message;
    }

    //Execution cost: 24505
    function test2() external view returns  (string memory){
         return message;
    }
}
```

## 19- There is no need to initialize variables with default values

If the variable is not set/initialized, it is assumed to have a default value (0, false, 0x0, etc., depending on the data type). If you explicitly initialize it with its default value, you are just wasting gas.

```solidity
uint256 foo = 0;//bad, expensive
uint256 bar;//good, cheap
```

## 20- Use short strings in revert, require checks

It is considered a best practice to append the error reason string with `require` statement. But these strings take up space in the deployed bytecode. Each reason string needs at least 32 bytes, so make sure your string complies with 32 bytes, otherwise it will become more expensive.

**Code Example:**

```solidity
require(balance >= amount, "Insufficient balance");//good
require(balance >= amount, "Too bad, it appears, ser you are broke, bye bye"); //bad
```

## 21- Use of Multiple require(s)

Using mutiple require statements is cheaper than using `&&` multiple check combinations. There are more advantages, such as easier to read code and better coverage reports.

```solidity
pragma solidity 0.8.10;

contract MultipleRequire {

    // Execution cost: 21723 gas
  function bad(uint a) public pure returns(uint256) {
        require(a>5 && a>10 && a>15 && a>20);
        return a;
    }

    // Execution cost: 21677 gas
    function good(uint a) public pure returns(uint256) {
        require(a>5);
        require(a>10);
        require(a>15);
        require(a>20);
        return a;
    }
}
```

## 22- Internal functions are cheaper to call

Set approriate function visibillities, as `internal` functions are cheaper to call than `public` functions.There is no need to mark a function as `public` if it is only meant to be called internally.

```d
//Following function will only be called interally
- function _func() public {...}
+ function _func() internal {...}
```

## 23- Save on data types

It is better to use `uint256` and `bytes32` than using uint8 for example. While it seems like `uint8` will consume less gas than uint256 it is not true, since the Ethereum virtual Machine(EVM) will still occupy `256` bits, fill 8 bits with the uint variable and fill the extra bites with zeros.

**Code Example:**

```solidity
pragma solidity ^0.8.1;

contract SaveGas {

    uint8 resulta = 0;
    uint resultb = 0;

    // Execution cost: 73446 gas
    function UseUint() external returns (uint) {
        uint selectedRange = 50;
        for (uint i=0; i < selectedRange; i++) {
            resultb += 1;
        }
        return resultb;
    }

    // Execution cost: 84175 gas
    function UseUInt8() external returns (uint8){
        uint8 selectedRange = 50;
        for (uint8 i=0; i < selectedRange; i++) {
            resulta += 1;
        }
        return resulta;
    }
}
```

## 24- Avoid repeated computations

Arithmetic computations cost gas, it is recommended to avoid repeated computations.

**Code Example:**

```solidity
// Unoptimized code:
for (uint i;i<length;) {
  tokens[i] += limit * price;

  unchecked {
    ++i
  }
}

// Optimized code:
uint local = limit * price;
for (uint i;i<length;) {
  tokens[i] += local;

  unchecked {
    ++i
  }
}
```

## 25- Single line swaps

One line to swap two variables without writing a function or temporary variable that needs more gas.

**Code Example:**

```solidity
(hello, world) = (world, hello);
```

## 26- Store data on memory not storage.

Choosing the perfect Data location is essential. You must know these:

- **storage** - variable is stored on the blockchain. It's a persistent state variable. Costs Gas to define it and change it.

- **memory** - temporary variable declared inside a function. No gas for declaring. But costs gas for changing memory variable (less than storage)

- **calldata** - like memory but non-modifiable and only available as an argument of external functions

Also, it is important to note that, If not specified data location, then it storage by default.

### Use `calldata` instead of `memory` for function parameters

```solidity
contract C {
    function add(uint[] memory arr) external returns (uint sum) {
        uint length = arr.length;
        for (uint i; i < arr.length; ) {
            sum += arr[i];

            unchecked {
              ++i
            }
        }
    }
}
```

In the above example, the dynamic array `arr` has the storage location `memory`. When the function gets called externally, the array values are kept in `calldata` and copied to memory during ABI decoding (using the opcode `calldataload` and `mstore`). And during the for loop, `arr[i]` accesses the value in memory using a `mload`. However, for the above example this is inefficient. Consider the following snippet instead:

```solidity
contract C {
    function add(uint[] calldata arr) external returns (uint sum) {
        uint length = arr.length;
        for (uint i; i < arr.length; ) {
            sum += arr[i];

            unchecked {
              ++i
            }
        }
    }
}
```

<!-- ## 29- Packing booleans

In Solidity, Boolean variables are stored as uint8 (unsigned integer of 8 bits). However, only 1 bit would be enough to store them. If you need up to 32 Booleans together, you can just follow the Packing Variables pattern. If you need more, you will use more slots than actually needed.

Pack Booleans in a single uint256 variable. To this purpose, create functions that pack and unpack the Booleans into and from a single variable. The cost of running these functions is cheaper than the cost of extra Storage. -->

## 32- Use Custom errors instead of revert strings

Custom errors from Solidity 0.8.4 are cheaper than revert strings (cheaper deployment cost and runtime cost when the revert condition is met)

**Code Example:**

```solidity

// Revert Strings
contract C {
    address payable owner;

    function withdraw() public {
        require(msg.sender == owner, "Unauthorized");

        //..
    }
}


//Custom errors
error Unauthorized();

contract C {
    address payable owner;

    function withdraw() public {
        if (msg.sender != owner) revert Unauthorized();

        //..
    }
}
```

## Credits

- https://dev.to/javier123454321/solidity-gas-optimizations-pt-3-packing-structs-23f4
- https://hacken.io/discover/solidity-gas-optimization/
- https://eip2535diamonds.substack.com/p/smart-contract-gas-optimization-with
- https://blog.birost.com/a?ID=00950-e4e25dc9-573d-4332-8262-41131961734f
- https://marduc812.com/2021/04/08/how-to-save-gas-in-your-ethereum-smart-contracts/
- https://betterprogramming.pub/how-to-write-smart-contracts-that-optimize-gas-spent-on-ethereum-30b5e9c5db85
- https://mudit.blog/solidity-gas-optimization-tips/
- http://www.cs.toronto.edu/~fanl/papers/gas-brain21.pdf
- https://gist.github.com/hrkrshnn/ee8fabd532058307229d65dcd5836ddc
- https://github.com/devanshbatham/Solidity-Gas-Optimization-Tips
- https://www.rareskills.io/post/gas-optimization
