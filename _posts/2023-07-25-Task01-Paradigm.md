---
layout: post
title: "Paradigm CTF WriteUps [Task - 01]"
date: 2023-07-25 09:07:16
categories: blog
---

Hi guys, I recently applied to a junior auditor role in a Blockchain Security company, I gonna share the insights I gained and tasks given by the team.

This is the first task,Where I need to solve some of the challenges in Paradigm CTF 2022.

You can find the other tasks here : [Task 2,3](https://d4r3-d3v1l.github.io/blog/2023/07/26/Task02-YUL-StackLimitEvader-Poc.html) [Task 4](https://d4r3-d3v1l.github.io/blog/2023/07/26/Task04-move-bytecode-poc.html)

<!--more-->

## OTTER-WORLD :

This is a sanity check challenge where we need to pass a number and call the function.

```rust
chall::cpi::get_flag(cpi_ctx, 0x1337 /* TODO */)?;
```

Place 0x7331 in place of TODO and run ./ruh.sh

```rust
chall::cpi::get_flag(cpi_ctx, 0x1337 * 0x7331)?;
```

## OTTERSWAP :

Here we have a swap functionality like from a to b and vice versa.
Initially the program has 2 pool accounts (pool_a and pool_b) and user_in_account and user_out_account.
It is calculating the out amount by a formula .

```
let x = in_pool_account.amount;
let y = out_pool_account.amount;
let out_amount = y - (x * y) / (x + amount);
```

It has a rounding issue which will result in lower balance in pools after the swap.
To know how much amount we need to send to get maximum tokens , I wrote a simple bruteforce function in python.

```python
def swap(pool_a , pool_b, user_in, user_out):
    res = None
    max_swap_amt = 0

    for user_in_amt in range(1,user_in+1):
        x = pool_a
        y = pool_b
        out_amount = y - (x * y) // (x + user_in_amt)
        user_in_mod = user_in - user_in_amt
        user_out_mod = user_out + out_amount
        pool_a_mod = pool_a + user_in_amt
        pool_b_mod = pool_b - out_amount

        for user_in_amt_mod in range(1,user_out_mod+1):
            x = pool_b_mod
            y = pool_a_mod
            # print(f" x and y {x} - {y} amt {user_in_amt_mod}")
            out_amount = y - (x * y) // (x + user_in_amt_mod)
            user_out_mod1 = user_out_mod - user_in_amt_mod
            user_in_mod1 = user_in_mod + out_amount
            pool_b_mod1 = pool_b_mod + user_in_amt_mod
            pool_a_mod1 = pool_a_mod - out_amount

            if user_in_mod1 + user_out_mod1 > max_swap_amt:
                max_swap_amt = user_in_mod1 + user_out_mod1
                res = (user_in_amt,user_in_amt_mod,pool_a_mod1,pool_b_mod1,user_in_mod1,user_out_mod1)

    return res

pool_a = 10
pool_b = 10
user_in = 10
user_out = 0
while True:
    user_in_amt , user_out_amt , pool_a,pool_b,user_in,user_out = swap(pool_a,pool_b,user_in,user_out)

    print(f"swap {user_in_amt} A to B")
    print(f"swap {user_out_amt} B to A")
    print(f"total amt -------------------   {user_in + user_out}")
    if user_in + user_out == 29:
        break
```

The output will have the amount and direction of the swaps.

```
swap 7 A to B
swap 3 B to A
total amt -------------------   12
swap 4 A to B
swap 3 B to A
total amt -------------------   14
swap 3 A to B
swap 2 B to A
total amt -------------------   16
swap 10 A to B
swap 3 B to A
total amt -------------------   19
swap 10 A to B
swap 2 B to A
total amt -------------------   22
swap 11 A to B
swap 1 B to A
total amt -------------------   29
```

We have only one cpi_ctx in the solve program and it is for a_to_b direction so we need to create another cpi_ctx for b_to_a.

By sending all the swaps like in the above output, you will have the 29 tokens.

## RANDOM :

This is a sanity check challenge where we need to pass the number 4 in the solve(uint256 guess) function .

## RESCUE :

The setup contract accidentally transferred 10 weth(ERC20Like) to the MasterChef contract.

Our goal is to drain the weth in the MasterChef contract :

```
return weth.balanceOf(address(mcHelper)) == 0;
```

The MasterChef contract has one external function `swapTokenForPoolToken`

Which takes in a token and a pool id .It swaps half of our token to tokenOut0 of the pool and tokenOut1 of the pool.

```solidity
_swap(tokenIn, tokenOut0, amountIn / 2);
_swap(tokenIn, tokenOut1, amountIn / 2);
```

After that it adds the tokenOut0 and tokenOut1 to the pool by using `_addLiquidity` Function.

### Bug:

There is issue with the `_addLiquidity` function , it is adding the all the amount in the contract of the token not the amount which is came from swaps.

```solidity
ERC20Like(token0).balanceOf(address(this)), ERC20Like(token1).balanceOf(address(this)),
```

So if one of the `tokenOut0` or `tokenOut1` is weth we can add the whole weth in the contract to the pool , this will drain all the weth in our contract .

Here we need to use a tokenIn which is other than the `tokenOut1` and `tokenOut0` and It should have enough pool to make swap.

So we can use dai token for this and 1 for poolId . poolId 1 have weth/usdc and weth to dai and weth to usdc pools also available .

First we deposit 20 ether to the weth . Then swaps 10 ether worth weth to usdc using router.

```solidity
path[0] = address(weth);
       path[1] = usdc;
       router.swapExactTokensForTokens(
           10 ether,
           0,
           path,
           address(mcHelper),
           block.timestamp
       );
```

This will add the 10 ether worth usdc to Masterchef contract.

Same way we swap weth for dai to our contract.

```solidity
      path[0] = address(weth);
      path[1] = dai;
       router.swapExactTokensForTokens(
           10 ether,
           0,
           path,
           address(this),
           block.timestamp
       );
```

Now we have 10ether worth DAI in our attacker contract . 10 ether worth WETH in MasterChef contract and 10ether worth USDC in MasterChef contract.

So if we call the swapTokenForPoolToken using poolId as 1 and token as DAI with the amount of the DAI ,

```solidity
mcHelper.swapTokenForPoolToken(1, dai, ERC20Like(dai).balanceOf(address(this)), 0);
```

This will drain the weth in the Masterchef contract.

## VANITY :

```solidity
 function solve(address signer, bytes memory signature) external {
        require(SignatureChecker.isValidSignatureNow(signer, MAGIC, signature), "Challenge/invalidSignature");

        solve(signer);
    }


  function solve(address who) private {
        uint score = 0;

        for (uint i = 0; i < 20; i++) if (bytes20(who)[i] == 0) score++;

        if (score > bestScore) bestScore = score;
    }
```

To solve this challenge we need to pass a signature check and the signer address should have 16 0s in it.

By checking the `isValidSignatureNow` function.

```solidity
  function isValidSignatureNow(
        address signer,
        bytes32 hash,
        bytes memory signature
    ) internal view returns (bool) {
        (address recovered, ECDSA.RecoverError error) = ECDSA.tryRecover(hash, signature);
        if (error == ECDSA.RecoverError.NoError && recovered == signer) {
            return true;
        }

        (bool success, bytes memory result) = signer.staticcall(
            abi.encodeWithSelector(IERC1271.isValidSignature.selector, hash, signature)
        );
        return (success && result.length == 32 && abi.decode(result, (bytes4)) == IERC1271.isValidSignature.selector);
    }
```

We can’t use a normal address here , and I found this site https://www.evm.codes/precompiled#0x02?fork=merge.

Look at the second condition where the contract making a staticcall to the signer and checking the result[4] bytes is equals to the isValidSignature.selector.
We can use the address 0x2 for this as it is a sha256 function. The staticcall will return the sha256 of the

```
 abi.encodeWithSelector(IERC1271.isValidSignature.selector, hash, signature)
```

If the 4 bytes of the result is equal to `isValidSignature.selector` It will return true.

We need to pass the valid signature to get the hash starting with isValidSignature.selector.

Wrote a python script but my laptop can’t withstand that bruteforce.

```python
import hashlib
!pip install eth_abi
from eth_abi import encode as encode_abi

def sha256(dat):
   dat = bytes.fromhex(dat)
   m = hashlib.sha256()
   m.update(dat)
   return m.digest().hex()

x = "1626ba7e"
hash_val = "19bb34e293bba96bf0caeea54cdd3d2dad7fdf44cbea855173fa84534fcfb528"

for i in range(1000000000):
   signature = str(i)
   data = encode_abi(['bytes4', 'bytes32', 'bytes'], [bytes.fromhex(x), bytes.fromhex(hash_val), signature.encode()])
   if sha256(data.hex()).startswith(x):
       print("Found matching signature: " + signature)
       break
```

## SOLHANA 1 :

This program allows users to deposit and withdraw tokens .
Lets check the functions

Setup_for_player - It create a state account and storing deposit account and deposit mint in it.

Deposit - it transfers amount of tokens from depositor account to deposit account and mint the same amount of voucher mint to the depositor_voucher_account .

Withdraw - It burns the voucher_mint tokens from the depositor_voucher_account and transfer the tokens to depositor_account from deposit_account.

During the initialization 10 bitcoin tokens are minted to sathosi’s account and later deposited in the deposit_account .

Our Win condition :

```
   pub async fn check_win(rpc: &RpcClient, player: &Pubkey) -> AnyResult<bool> {
       let program_id = Keypair::from_bytes(&PROGRAM_KEY)?.pubkey();
       let (deposit_account, _) = Pubkey::find_program_address(&[player.as_ref(), TOKEN], &program_id);

       let account_data = rpc.get_account_data(&deposit_account).await?;
       let account = Account::unpack(&account_data)?;

       Ok(account.amount == 0)
   }
```

So we need to steal the 10 bitcoin tokens from the `deposit_account`.

But During initialization of the challenge there created a bitcoin account for player and mints 10 bitcoin to player bitcoin account.

If you see carefully the `voucher_mint` account is created for the player but didn’t stored anywhere and it has no mint .

If we want to steal the tokens from deposit account we must have voucher_mint token in our `depositor_voucher_mint` account to burn .

Can we pass an arbitrary `depositor_voucher_account` while withdrawing?

Account constraints :

```
#[account(mut, constraint = voucher_mint.mint_authority == COption::Some(state.key()))]
   pub voucher_mint: Account<'info, Mint>,
   #[account(mut, constraint = depositor_account.mint == state.deposit_mint)]
   pub depositor_account: Account<'info, TokenAccount>,
   #[account(mut, constraint = depositor_voucher_account.mint == voucher_mint.key())]
   pub depositor_voucher_account: Account<'info, TokenAccount>,
```

If we pass an arbitrary depositor_voucher_account , the mint of our account is to be equal to the voucher_mint.

To pass this check we create a fake voucher_mint and mint 10 tokens to the depositor_voucher_account.

Bug :
But the voucher_mint has a constraint that the mint_authority should be equal to the state.
In solana we can change the mint_authority of a token by the SetAuthority function ,so after minting tokens we set the mint_authority to the state.

Exploit :  
Create a fake mint token and fake mint account `(depositoe_voucher_account)` and mint 10 tokens.
Set the mint authority to state account .

Now call the withdraw instruction with `voucher_mint` as fake_mint and `depositor_voucher_account` as `fake_mint_account` and `depositor_account` as player’s bitcoin account to steal the amount from `deposit_account`.

## SOLHANA 2 :

This program allows user to swap tokens (wo_eth,so_eth,st_eth).

Lets check the functions :

Deposit - Transfers tokens from depositor_account to pool_account and mints voucher_mint to
`depositor_voucher_account`.

Withdraw - Burns voucher_mint from the `depositor_voucher_account` and transfers the tokens from the `pool_account` to `depositor_account`.

Swap - Swaps the tokens , it calculates the to_amount and transfers the tokens from `to_pool_account` to `to_swapper` and `from_amout` to `from_pool_account` from `from_swapper` account.

During the Initialization of the challenge , it creates 3 pools for each wo_eth,so_eth and st_eth and mints 100_000_000 to it .
It also creates a token account for each mint for player and mints 1000 each tokens to it.

Our win condition :

```
Ok(wo_account.amount + so_account.amount + st_account.amount < 150_000_000)
```

We need to steal half of the amount in the pool to solve this challenge.

The deposit and withdraw are the same as before .so I checked the swap function .

It is calculating the `to_amount` from `from_amount` thought of any rounding issues like otterswap but it is checking the amount again by `check_amount` and throwing an error if `check_amount ! = from_amount`.

```rust
let to_amount = u64::try_from(
           u128::from(from_amount).checked_mul(10_u128.pow(ctx.accounts.to_pool.decimals.into())).unwrap().checked_div(10_u128.pow(ctx.accounts.from_pool.decimals.into())).unwrap()
       ).unwrap();
```

In the calculation it is using `to_pool.decimals` , if you see the challenge.rs the decimals are different for tokens . wo_eth - 8 , so_eth - 6 and st_eth - 8.

By this if we swap so_eth to wo_eth or st_eth we get \*100 times . But we have only 1000 tokens of each in our player token accounts.

After looking at the code again and again for some time , I saw that deposit may help me .

### Bug :

There is no check for deposit_mint , so we can pass a depositor account of wo_eth and mint so_eth `voucher_mint` to `depositor_voucher_account` by providing the `deposit_mint` as so_eth.

Then withdraw the so_eth tokens by burning the so_eth `voucher_mint`.

In this way we will have more so_eth tokens and swaps it for higher returns.

### Exploit :

We have 1000 of each wo_eth, so_eth and st_eth.

First deposit 1000 wo_eth and mint 1000 so_eth voucher tokens .

Withdraw the 1000 so_eth voucher tokens to 1000 so_eth tokens .

We now have 2000 so_eth tokens .

If we swap the 2000 so_eth tokens for wo_eth tokens , we will have 2000\*100 tokens of wo_eth.

The pool will have 99800000 of wo_eth tokens ,if we do the same for those 200000 wo_eth tokens we can reduce the pool's balance and win the challenge .

## SOLHANA 3 :

This program implementing a flashloan for a atomcoin token.

Lets check the functions :

Deposit and withdraw are same as before and also in the readme of the challenge we have  
“i will be nice and say you do not need to use deposit or withdraw here, only borrow and repay”.

Borrow - Transfers the amount `from pool_account` to `depositor_account` with some checks.

Repay - Transfers the same amount we borrowed to the `pool_account` from the `depositor_account`.

Constraints :
Borrow and Repay instructions must be in the same transaction.
Checks if the Repay instruction have the same amount of we borrowed
If this checks passes it will initiate a borrow.

Win condition :

```rust
 pub async fn check_win(rpc: &RpcClient, player: &Pubkey) -> AnyResult<bool> {
       let program_id = Keypair::from_bytes(&PROGRAM_KEY)?.pubkey();
       let (pool_account, _) = Pubkey::find_program_address(&[player.as_ref(), TOKEN], &program_id);

       let account_data = rpc.get_account_data(&pool_account).await?;
       let account = Account::unpack(&account_data)?;

       Ok(account.amount <= 2)
   }
```

The pool has 100 atomcoin tokens we must steal atleast 98 tokens to win .

I thought of borrowing 2 times and repaying one time but there is a flag which won’t allow us to borrow second time before repaying the previous one.

```
token::transfer(transfer_ctx, amount)?;
ctx.accounts.pool.borrowing = true;
```

Wondering how to skip the repay , then I saw this in the readme .
“the one exception is if you should just so happen to decide you need to deploy your own program to complete a challenge (this is a hint). “

### Bug :

Deploying a own program ? Yes we can use an external program to call repay instruction using cpi .

### Exploit :

Create a program to make repay instruction . This allows us to make another borrow

```
borrow-50 ,repay-0(from cpi) , borrow-50 , repay-50.

depositor_account  - 50
n condition :

borrow -25, repay-0(from cpi) , borrow -25, repay-25 .

Depositor_account - 75

borrow -12, repay-0(from cpi) , borrow -12, repay-12 .

Depositor_account - 87

Like this for 6 , 3 and 2 the depositor_account will have 98 tokens .

```
