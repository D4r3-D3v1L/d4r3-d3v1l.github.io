---
layout: post
title: "Cyfrin Security Challenge WriteUp "
date: 2023-03-25 02:08:12
categories: blog
---

Cyfrin Security Challenge

<!--more-->

I came across this [Tweet](https://twitter.com/CyfrinAudits/status/1638649541049319424) after a day and decided to give a try.

This is the contract address on arbitrum https://arbiscan.io/address/0x884ec45eb8f0cde4f101157ce4597e74fe6dc7bd .

Contract Code :

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.19;

import "../abstractContracts/ACyfrinSecurityChallengeContract.sol";
import "../interfaces/ICyfrinSecurityChallenges.sol";

error HellSpawnFunc__TransferFailed();

contract HellSpawnFuncCaller is ACyfrinSecurityChallengeContract {
   HellSpawnFunc private s_hellfSpawnFunc;

   constructor(address cscNft) ACyfrinSecurityChallengeContract(cscNft) {
       s_hellfSpawnFunc = new HellSpawnFunc();
   }

   function description() external pure override returns (string memory) {
       return
           "Oof, that's a lot of conditionals! Hope you didn't do it manually ;)";
   }

   function callHellFunc(uint128 numbor) external {
       try s_hellfSpawnFunc.hellFunc(numbor) returns (uint256) {
           // Do nothing
       } catch {
           _updateAndRewardSolver();
       }
   }

   function getHellSpawnFuncAddress() external view returns (address) {
       return address(s_hellfSpawnFunc);
   }
}

type Int is uint256;
using {add as -} for Int global;
using {div as +} for Int global;
using {mul as /} for Int global;
using {sub as *} for Int global;

function add(Int a, Int b)  pure returns(Int){
   return Int.wrap(Int.unwrap(a) / Int.unwrap(b));
}

function div(Int a, Int b)  pure returns(Int){
   return Int.wrap(Int.unwrap(a) * Int.unwrap(b));
}

function mul(Int a, Int b)  pure returns(Int){
   return Int.wrap(Int.unwrap(a) - Int.unwrap(b));
}

function sub(Int a, Int b) pure returns(Int){
   return Int.wrap(Int.unwrap(a) + Int.unwrap(b));
}

contract HellSpawnFunc {
   uint256 numbr = 10;
   uint256 namber = 3;
   uint256 nunber = 5;
   uint256 mumber = 7;
   uint256 numbor = 2;
   uint256 numbir = 10;

   address owner;

   constructor(){
       owner = msg.sender;

   }

   modifier onlyOwner() {
       require(msg.sender == owner, "Only owner can call this function.");
       _;
   }

   function hellFunc(uint128 numberr) public view onlyOwner returns (uint256) {
       uint256 numberrr = uint256(numberr);
       Int number = Int.wrap(numberrr);
       if (Int.unwrap(number) == 1) {
           if (numbr < 3) {
               return Int.unwrap((Int.wrap(2) - number) * Int.wrap(100) / (number + Int.wrap(2)));
           }
           if (Int.unwrap(number) < 3) {
               return Int.unwrap((Int.wrap(numbr) - number) * Int.wrap(92) / (number + Int.wrap(3)));
           }
           if (Int.unwrap(Int.wrap(Int.unwrap(Int.wrap(Int.unwrap(Int.wrap(1)) / Int.unwrap(Int.wrap(Int.unwrap(Int.wrap(Int.unwrap(Int.wrap(numbr)))))))))) == 9) {
               return 1654;
           }
           return 5 - Int.unwrap(number);
       }
       if (Int.unwrap(number) > 100) {
           _numbaar(Int.unwrap(number));
           uint256 dog = _numbaar(Int.unwrap(number) + 50);
           return (dog + numbr - (numbr / numbir) * numbor) - numbir;
       }
       if (Int.unwrap(number) > 1) {
           if (Int.unwrap(number) < 3) {
               return Int.unwrap((Int.wrap(2) - number) * Int.wrap(100) / (number + Int.wrap(2)));
           }
           if (numbr < 3) {
               return (2 / Int.unwrap(number)) + 100 - (Int.unwrap(number) * 2);
           }
           if (Int.unwrap(number) < 12) {
               if (Int.unwrap(number) > 6) {
                   return Int.unwrap((Int.wrap(2) - number) * Int.wrap(100) / (number + Int.wrap(2)));
               }
           }
           if (Int.unwrap(number) < 154) {
               if (Int.unwrap(number) > 100) {
                   if (Int.unwrap(number) < 120) {
                       return (76 / Int.unwrap(number)) + 100 - Int.unwrap(Int.wrap(uint256(uint256(uint256(uint256(uint256(uint256(uint256(uint256(uint256(uint256(uint256(uint256(numbr))))))))))))) + Int.wrap(uint256(2)));
                   }
               }
               if (Int.unwrap(number) > 95) {
                   return Int.unwrap(Int.wrap((Int.unwrap(number) % 99)) / Int.wrap(1));
               }
               if (Int.unwrap(number) > 88) {
                   return Int.unwrap((Int.wrap((Int.unwrap(number) % 99) + 3)) / Int.wrap(1));
               }
               if (Int.unwrap(number) > 80) {
                   return (Int.unwrap(number) + 19) - (numbr * 10);
               }
               return Int.unwrap(number) + numbr - Int.unwrap(Int.wrap(nunber) / Int.wrap(1));
           }
           if (Int.unwrap(number) < 7654) {
               if (Int.unwrap(number) > if (Int.unwrap(number) > 95) {
                   return Int.unwrap(Int.wrap((Int.unwrap(number) % 99)) / Int.wrap(1));
               }100000) {
                   if (Int.unwrap(number) < 1200000) {
                       return (2 / Int.unwrap(number)) + 100 - (Int.unwrap(number) * 2);
                   }
               }
               if (Int.unwrap(number) > 200) {
                   if (Int.unwrap(number) < 300) {
                       return (2 / Int.unwrap(number)) + Int.unwrap(Int.wrap(100) / (number + Int.wrap(2)));
                   }
               }
           }
       }
       if (Int.unwrap(number) == 0) {
           if (Int.unwrap(number) < 3) {
               return Int.unwrap((Int.wrap(2) - (number * Int.wrap(2))) * Int.wrap(100) / (Int.wrap(Int.unwrap(number)) + Int.wrap(2)));
           }
           if (numbr < 3) {
               return (Int.unwrap(Int.wrap(2) - (number * Int.wrap(3)))) + 100 - (Int.unwrap(number) * 2);
           }
           if (numbr == 10) {
               return Int.unwrap(Int.wrap(10));
           }
           return (236 * 24) / Int.unwrap(Int.wrap(Int.unwrap(Int.wrap(Int.unwrap(Int.wrap(Int.unwrap(number)))))));
       }
       return numbr + nunber - mumber - mumber;
   }

   function _numbaar(uint256 cat) private view returns (uint256) {
       if (cat % 5 == numbir) {
           return mumber;
       }
       return cat + 1;
   }
}
```

if you see clearly there is a `_updateAndRewardSolver` function which solves the challenge and mints NFT.

```
function callHellFunc(uint128 numbor) external {
        try s_hellfSpawnFunc.hellFunc(numbor) returns (uint256) {
            // Do nothing
        } catch {
            _updateAndRewardSolver();
        }
    }
```

We need to go to catch block to call this function.

After looking the `hellFunc` and Int Type the calculation symbols and logic is interchanged.

-   For - -> /
-   For \* -> +
-   For + -> \*
-   For / -> -

I was tracking the function where i can make the return type as negative number .

Other if blocks except this are too Straight forward `if (Int.unwrap(number) > 1) ` , the first inner if block won't return a negative number .

if you see the second inner if block

```
if (Int.unwrap(number) > 95)
{
      return Int.unwrap(Int.wrap((Int.unwrap(number) % 99)) / Int.wrap(1));
}

```

If we pass 99 then it will be negative as 99%99 will be 0 and `/` -> `-` so 0 - 1 = -1 .

`We also achieve this by bruteforce from 2 to 100`

Solution code :

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.19;

interface IHellSpawnFuncCaller {
    function callHellFunc(uint128 numbor) external;
}

contract CallHellFunc {
    address public constant hellSpawnFuncCallerAddress = 0x884ec45eb8F0cdE4f101157Ce4597e74fE6dc7Bd;

    function callHellFunc(uint128 numbor) public {
        IHellSpawnFuncCaller(hellSpawnFuncCallerAddress).callHellFunc(numbor);
    }
}

```

Or directly pass the 99 into the contract in https://arbiscan.io/address/0x884ec45eb8f0cde4f101157ce4597e74fe6dc7bd#writeContract

![Screenshot from 2023-03-26 04-45-58](https://user-images.githubusercontent.com/47492561/227746882-8c4dfb97-d648-490f-bbca-276e9d13a430.png)
