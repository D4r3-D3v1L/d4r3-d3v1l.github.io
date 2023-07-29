---
layout: post
title: "Understanding YUL Optimization Passes [Task - 02,03]"
date: 2023-07-26 08:07:34
categories: blog
---

This is the second task of the interview process which consists of YUL language , Optimization Passes and memory-safe assembly in solidity.

<!--more-->

After the [First task](https://d4r3-d3v1l.github.io/blog/2023/07/25/Task01-Paradigm.html) , the recruiter gave me a week to learn YUL language which is intermediate language for EVM.

I have refered these resources [YUL basics ](https://calnix.gitbook.io/eth-dev/yul/yul),[YUL Documentation](https://docs.soliditylang.org/en/latest/yul.html) .

After one week , He asked me about `StackLimitEvader` and how it works.

### StackLimitEvader ?

StackLimitEvader is a low-level optimization pass used in Yul to prevent `stack-too-deep` errors. These errors occur when there are too many variables on the stack at once.

However, this transformation is only enabled in the absence of inline assembly for now, since there is no way to indicate that inline assembly will respect Solidity’s memory model, which is a prerequisite for the transformation to be valid.

To enable the StackLimitEvader, Yul code needs to indicate that it respects an area of reserved memory. This can be done with the [**memoryguard**](https://docs.soliditylang.org/en/latest/yul.html?#memoryguard) function.

If you want to skip the code flow of How StackLimitEvader pass works Please click here [Example](#example)

### How the StackLimitEvader pass work.

[code](https://github.com/ethereum/solidity/blob/29751849b58a7db5eec3f297087544ae9531f0df/libyul/optimiser/StackLimitEvader.cpp)

![Flow](https://file.notion.so/f/s/e294a517-c867-47c1-87b7-702addc6b695/Untitled.png?id=fa89efdb-8e6f-4cb3-9584-f7fe1dce360e&table=block&spaceId=ab48cbd9-e93c-449f-912e-ad13ad550a0e&expirationTimestamp=1690588800000&signature=7H5-vyw6SrLu-mJvHD_Bz6rwiBq-No_OIjDKPvzRGJ4&downloadName=Untitled.png)

`Suite.cpp` file which executes various optimization steps, including the `StackLimitEvader` technique. The `StackLimitEvader::run` function is called, which in turn calls two other `run` functions in the `StackLimitEvader.cpp` file.

**Suite.cpp → calls StackLimitEvader(*context,*object) → Move the Stack to Memory.**

Here's how the `StackLimitEvader` pass is called:

```c++
// Run the user-supplied clean up sequence
	suite.runSequence(_optimisationCleanupSequence, ast);
	// Hard-coded FunctionGrouper step is used to bring the AST into a canonical form required by the StackCompressor
	// and StackLimitEvader. This is hard-coded as the last step, as some previously executed steps may break the
	// aforementioned form, thus causing the StackCompressor/StackLimitEvader to throw.
	suite.runSequence("g", ast);

	if (evmDialect)
	{
		yulAssert(_meter, "");
		ConstantOptimiser{*evmDialect, *_meter}(ast);
		if (usesOptimizedCodeGenerator)
		{
			StackCompressor::run(
				_dialect,
				_object,
				_optimizeStackAllocation,
				stackCompressorMaxIterations
			);
			**if (evmDialect->providesObjectAccess())
				StackLimitEvader::run(suite.m_context, _object);
		}
		else if (evmDialect->providesObjectAccess() && _optimizeStackAllocation)
			StackLimitEvader::run(suite.m_context, _object);**
	}
```

In this Suite.cpp If the EVM dialect is set (i.e., the target platform is EVM) some checks and If the `usesOptimizedCodeGenerator` flag is set, the `StackCompressor` is applied to optimize the use of the stack in the Yul code. If the EVM dialect supports object access (i.e., the `providesObjectAccess`() function returns true), the `StackLimitEvader` it takes the `m_context` object and the `Object` object as arguments. The `m_context` object provides the context in which the `StackLimitEvader` pass will run, while the `Object` object represents the Yul code that has been optimized by previous optimization passes.

`StackLimitEvader.cpp :`

Here's the entry point to the `StackLimitEvader` pass:

```c++
void StackLimitEvader::run(
	OptimiserStepContext& _context,
	Object& _object
)
{
	auto const* evmDialect = dynamic_cast<EVMDialect const*>(&_context.dialect);
	yulAssert(
		evmDialect && evmDialect->providesObjectAccess(),
		"StackLimitEvader can only be run on objects using the EVMDialect with object access."
	);
	if (evmDialect && evmDialect->evmVersion().canOverchargeGasForCall())
	{
		yul::AsmAnalysisInfo analysisInfo = yul::AsmAnalyzer::analyzeStrictAssertCorrect(*evmDialect, _object);
		unique_ptr<CFG> cfg = ControlFlowGraphBuilder::build(analysisInfo, *evmDialect, *_object.code);
		run(_context, _object, StackLayoutGenerator::reportStackTooDeep(*cfg));
	}
	else
		run(_context, _object, CompilabilityChecker{
			_context.dialect,
			_object,
			true
		}.unreachableVariables);

}
```

It checks if the input object uses the EVMDialect with object access, and if the EVM version used in the dialect supports overcharging gas for call instructions.

If both conditions are satisfied :

-   It calls the 2nd run method with the same context , object and `StackLayoutGenerator::reportStackTooDeep(\*cfg)`
-   reportStackTooDeep :
    this method takes a Control Flow Graph (`CFG`) object as input and returns a map of Yul function names to a vector of `StackToDeep` objects. Each `StackToDeep` object represents a point in the function where the stack depth exceeds the maximum allowed depth

`This will call the 2nd run method :`

```c++
void StackLimitEvader::run(
	OptimiserStepContext& _context,
	Object& _object,
	map<YulString, vector<StackLayoutGenerator::StackTooDeep>> const& _stackTooDeepErrors
)
{
	map<YulString, set<YulString>> unreachableVariables;
	for (auto&& [function, stackTooDeepErrors]: _stackTooDeepErrors)
		// TODO: choose wisely.
		for (auto const& stackTooDeepError: stackTooDeepErrors)
			unreachableVariables[function] += stackTooDeepError.variableChoices | ranges::views::take(stackTooDeepError.deficit) | ranges::to<set<YulString>>;
	run(_context, _object, unreachableVariables);
}
```

This code is adding to the `unreachableVariables` map the set of variables that are unreachable due to the stack being too deep. The set of variables to add is determined based on the `variableChoices` set of each `StackTooDeep` object in the `\_stackTooDeepErrors` map

and then calls `3rd run method.`

Else block :

-   It calls the 3rd run method with the same context , object and `CompilabilityChecker{\_context.dialect,\_object,true}.unreachableVariables.` which is used to identify the unreachable variables.
-   CompilabilityChecker :
    The CompilabilityChecker takes the dialect, object, and a boolean parameter as input. The boolean parameter specifies whether to run a simulation of the execution of the program or not. If it is set to true, the CompilabilityChecker will run the simulation and compute which parts of the code are unreachable.
    The `unreachableVariables` member function of the CompilabilityChecker returns a map that associates each function name with a set of unreachable variable names. These variable names represent the local variables that are never accessed in the function because the corresponding code paths are unreachable. This information is used by the `StackLimitEvader` to modify the Yul code and remove these unused variables from the stack.

`3rd run method`

```c++
void StackLimitEvader::run(
	OptimiserStepContext& _context,
	Object& _object,
	map<YulString, set<YulString>> const& _unreachableVariables
)
{
	yulAssert(_object.code, "");
	auto const* evmDialect = dynamic_cast<EVMDialect const*>(&_context.dialect);
	yulAssert(
		evmDialect && evmDialect->providesObjectAccess(),
		"StackLimitEvader can only be run on objects using the EVMDialect with object access."
	);

	vector<FunctionCall*> memoryGuardCalls = FunctionCallFinder::run(
		*_object.code,
		"memoryguard"_yulstring
	);
	// Do not optimise, if no ``memoryguard`` call is found.
	if (memoryGuardCalls.empty())
		return;

	// Make sure all calls to ``memoryguard`` we found have the same value as argument (otherwise, abort).
	u256 reservedMemory = literalArgumentValue(*memoryGuardCalls.front());
	yulAssert(reservedMemory < u256(1) << 32 - 1, "");

	for (FunctionCall const* memoryGuardCall: memoryGuardCalls)
		if (reservedMemory != literalArgumentValue(*memoryGuardCall))
			return;

	CallGraph callGraph = CallGraphGenerator::callGraph(*_object.code);

	// We cannot move variables in recursive functions to fixed memory offsets.
	for (YulString function: callGraph.recursiveFunctions())
		if (_unreachableVariables.count(function))
			return;

	map<YulString, FunctionDefinition const*> functionDefinitions = allFunctionDefinitions(*_object.code);

	MemoryOffsetAllocator memoryOffsetAllocator{_unreachableVariables, callGraph.functionCalls, functionDefinitions};
	uint64_t requiredSlots = memoryOffsetAllocator.run();
	yulAssert(requiredSlots < (uint64_t(1) << 32) - 1, "");

	StackToMemoryMover::run(_context, reservedMemory, memoryOffsetAllocator.slotAllocations, requiredSlots, *_object.code);

	reservedMemory += 32 * requiredSlots;
	for (FunctionCall* memoryGuardCall: FunctionCallFinder::run(*_object.code, "memoryguard"_yulstring))
	{
		Literal* literal = std::get_if<Literal>(&memoryGuardCall->arguments.front());
		yulAssert(literal && literal->kind == LiteralKind::Number, "");
		literal->value = YulString{toCompactHexWithPrefix(reservedMemory)};
	}
}
```

Inputs : OptimiserStepContext, an Object, and a map of unreachablevariables

This is the main logic of the code ,lets break down :

-   First there are 2 assertions regarding object.code and providesObjectAccess .
-   It then finds all `memoryguard` function calls in the Yul code and if there is no memoryguard calls it returns.
-   If there are `memoryguard` calls, it checks that all calls use the same value as an argument and not more than 2^32 - 1
-   It then constructs a call graph of the Yul code and checks that there are no recursive functions that have unreachable variables
-   Then we create the `MemoryOffsetAllocator` with the context,functioncalls and functiondefinitions.

    ```c++
    struct MemoryOffsetAllocator
    {
    	uint64_t run(YulString _function = YulString{})
    	{
    		if (slotsRequiredForFunction.count(_function))
    			return slotsRequiredForFunction[_function];

    		// Assign to zero early to guard against recursive calls.
    		slotsRequiredForFunction[_function] = 0;

    		uint64_t requiredSlots = 0;
    		if (callGraph.count(_function))
    			for (YulString child: callGraph.at(_function))
    				requiredSlots = std::max(run(child), requiredSlots);

    		if (auto const* unreachables = util::valueOrNullptr(unreachableVariables, _function))
    		{
    			if (FunctionDefinition const* functionDefinition = util::valueOrDefault(functionDefinitions, _function, nullptr, util::allow_copy))
    				if (
    					size_t totalArgCount = functionDefinition->returnVariables.size() + functionDefinition->parameters.size();
    					totalArgCount > 16
    				)
    					for (TypedName const& var: ranges::concat_view(
    						functionDefinition->parameters,
    						functionDefinition->returnVariables
    					) | ranges::views::take(totalArgCount - 16))
    						slotAllocations[var.name] = requiredSlots++;

    			// Assign slots for all variables that become unreachable in the function body, if the above did not
    			// assign a slot for them already.
    			for (YulString variable: *unreachables)
    				// The empty case is a function with too many arguments or return values,
    				// which was already handled above.
    				if (!variable.empty() && !slotAllocations.count(variable))
    					slotAllocations[variable] = requiredSlots++;
    		}

    		return slotsRequiredForFunction[_function] = requiredSlots;
    	}

    	/// Maps function names to the set of unreachable variables in that function.
    	/// An empty variable name means that the function has too many arguments or return variables.
    	map<YulString, set<YulString>> const& unreachableVariables;
    	/// The graph of immediate function calls of all functions.
    	map<YulString, set<YulString>> const& callGraph;
    	/// Maps the name of each user-defined function to its definition.
    	map<YulString, FunctionDefinition const*> const& functionDefinitions;

    	/// Maps variable names to the memory slot the respective variable is assigned.
    	map<YulString, uint64_t> slotAllocations{};
    	/// Maps function names to the number of memory slots the respective function requires.
    	map<YulString, uint64_t> slotsRequiredForFunction{};
    };
    ```

-   This structs also implements a `run` method which rerturns the number of memory slots required by each Yul function
    -   `unreachableVariables` : A map that maps function names to the set of unreachable variables in those functions.
    -   `callGraph` : A map that represents the call graph of the contract, mapping each function name to the set of functions that it calls directly.
    -   `functionDefinitions` : A map that maps function names to their definitions.
    -   `slotAllocations` : A map that maps variable names to the memory slot the respective variable is assigned.
    -   `slotsRequiredForFunction` : A map that maps function names to the number of memory slots the respective function requires.
-   First checks if the number of required memory slots for the given function is already known by checking if the function name is present in the `slotsRequiredForFunction` map. If the number of required slots is already known, the method simply returns the value from the map.
-   If the number of required slots is not known, the method calculates it recursively by traversing the call graph of the function and computing the maximum number of slots required by any of its child functions.
-   it then retrieve the value of the `unreachableVariables` map corresponding to the key `_function`. If the key exists in the map, the function will return a pointer to its value. If the key does not exist, the function will return a null pointer.
-   if the total number of arguments and return variables is greater than 16, then some of them need to be stored on the stack .

Finally returns the `requiredSlots .`

-   The function then calls `run` in `StackToMemoryMover` , `which moves all stack variables to memory based on the offsets` generated by the `MemoryOffsetAllocator .`
-   It then updates the `memoryguard` function call with the new value of reserved memory

### Example

`function_args.yul`

```jsx
{
    {
        mstore(0x40, memoryguard(128))
        sstore(0, f(0))
    }
    function f(a1) -> v {
	let a2 := calldataload(mul(2,4))
	let a3 := calldataload(mul(3,4))
	let a4 := calldataload(mul(4,4))
	let a5 := calldataload(mul(5,4))
	let a6 := calldataload(mul(6,4))
	let a7 := calldataload(mul(7,4))
	let a8 := calldataload(mul(8,4))
	let a9 := calldataload(mul(9,4))
	let a10 := calldataload(mul(10,4))
	let a11 := calldataload(mul(11,4))
	let a12 := calldataload(mul(12,4))
	let a13 := calldataload(mul(13,4))
	let a14 := calldataload(mul(14,4))
	let a15 := calldataload(mul(15,4))
	let a16 := calldataload(mul(16,4))
	let a17 := calldataload(mul(17,4))
	sstore(0, a1)
	sstore(mul(17,4), a17)
	sstore(mul(16,4), a16)
	sstore(mul(15,4), a15)
	sstore(mul(14,4), a14)
	sstore(mul(13,4), a13)
	sstore(mul(12,4), a12)
	sstore(mul(11,4), a11)
	sstore(mul(10,4), a10)
	sstore(mul(9,4), a9)
	sstore(mul(8,4), a8)
	sstore(mul(7,4), a7)
	sstore(mul(6,4), a6)
	sstore(mul(5,4), a5)
	sstore(mul(4,4), a4)
	sstore(mul(3,4), a3)
	sstore(mul(2,4), a2)
	sstore(mul(1,4), a1)
    }
}

```

**Without optimization**

```bash
solc --strict-assembly function_args.yul
```

```bash
Uncaught exception:
/solidity/libyul/backends/evm/EVMObjectCompiler.cpp(126): Throw in function void solidity::yul::EVMObjectCompiler::run(solidity::yul::Object&, bool)
Dynamic exception type: boost::wrapexcept<solidity::yul::StackTooDeepError>
std::exception::what: **Variable a1 is 2 slot(s) too deep inside the stack. Stack too deep.** Try compiling with `--via-ir` (cli) or the equivalent `viaIR: true` (standard JSON) while enabling the optimizer. Otherwise, try removing local variables.
[solidity::util::tag_comment*] = Variable a1 is 2 slot(s) too deep inside the stack. Stack too deep. Try compiling with `--via-ir` (cli) or the equivalent `viaIR: true` (standard JSON) while enabling the optimizer. Otherwise, try removing local variables.
```

**using --optimizer will use the StackLimitEvader and give us the optimized code.**

```bash
solc --strict-assembly --optimize function_args.yul
```

```bash
object "object" {
    code {
        {
            sstore(68, calldataload(68))
            sstore(0x40, calldataload(0x40))
            sstore(60, calldataload(60))
            sstore(56, calldataload(56))
            sstore(52, calldataload(52))
            sstore(48, calldataload(48))
            sstore(44, calldataload(44))
            sstore(40, calldataload(40))
            sstore(36, calldataload(36))
            sstore(32, calldataload(32))
            sstore(28, calldataload(28))
            sstore(24, calldataload(24))
            sstore(20, calldataload(20))
            sstore(16, calldataload(16))
            sstore(12, calldataload(12))
            sstore(8, calldataload(8))
            sstore(4, 0)
            sstore(0, 0)
        }
    }
}
```

## Task -03

Recruiter asked me `can you provide an example of code that is normally safe, but becomes unsafe after a compiler optimization pass?`.

**Initial Thoughts**

-   Normally since solidity follows the memory rules ,optimization over that code won’t cause any issues .

-   Instead assembly gives a good flexibility for developers over opcodes and memory operations ,so by default optimizer consider it as memory unsafe and won’t do any optimizations.

-   But if the developer specify explicitly using `memory-safe` annotation , the compiler simply assumes it as memory safe and do optimizations but the compiler cannot confirm or validate its memory safety.

-   so if the memory unsafe assembly is specified as memory safe , the optimizations which moves the values from stack to memory to avoid `stack deep error` might result in delivering improper values to the variables which result in undefined behaviour.

-   But the optimizer moves the stack variables to the memory only if it encounters a `stack too deep error` in the code . So a normally safe code with stack too deep error `won’t compile` without a optimizer to becomes a unsafe after the optimization pass.

I told him that It is not possible to give such example. But he gave me a hint that it is possible .

### Building Example :

##### After going through all the optimization passes ,Here is a brief summary of what happens in each pass:

1. `DeadAssignmentEliminator`: This pass eliminates unnecessary assignments to variables that are never read from again. This reduces the amount of unnecessary code in the program.
2. `FullInliner`: This pass inlines small functions into the call site to reduce the overhead of function calls.
3. `JumpCleaner`: This pass removes unnecessary jump instructions in the code.
4. `UnusedFunctionRemover`: This pass removes unused functions from the code. This reduces the size of the program.
5. `UnusedVariableRemover`: This pass removes unused variables from the code. This reduces the size of the program.
6. `BlockSimplifier`: This pass simplifies blocks of code that can be reduced to a single instruction.
7. `StackHeightOptimizer`: This pass replaces duplicate stack operations with a single instruction. This reduces the amount of stack usage.
8. `ConstantFolder`: This pass evaluates constant expressions at compile time and replaces them with their values. This reduces the number of instructions executed at runtime.
9. `VariableReplacer`: This pass replaces local variables with their values when possible. This reduces the number of memory accesses and improves performance.
10. `StackLimitEvader`: This pass optimizes the use of the EVM stack by moving variables to memory when necessary. This reduces the risk of running out of stack space during execution.

Our requirement for Example :

1. produces a result that is not a compilation error when run normally
2. produces a different result when optimized

### Solution :

We should write a code which should increase the stack size while doing the optimization ,so that it can trigger the `StackLimitEvader` pass to overwrite the values in the memory.

But the actual question here is which optimization pass can do this , as you can see the above passes you may have already got the answer ,Yes.. [FullInliner](https://docs.soliditylang.org/en/latest/internals/optimizer.html#fullinliner).

The flow look like this :
![Flow](https://file.notion.so/f/s/eb4e1ba3-a424-46cc-a822-157d7e14d31c/Untitled_drawing.jpg?id=ead7bb32-8fbc-4fb0-8f52-9e956dfc6882&table=block&spaceId=ab48cbd9-e93c-449f-912e-ad13ad550a0e&expirationTimestamp=1690596000000&signature=yeMo74ragAVuMdGETTbw3l2mEPhL189XyRGUiJMPJaU&downloadName=Untitled+drawing.jpg)

Code :

```jsx
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;
contract Exp5 {
    uint256[20] numbers = [1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20];
    function stackTooDeep() public view returns (uint256) {
        uint256 _a;
        uint256 _b;
        uint256 _c;
        uint256 _d;
        uint256 _e;
        uint256 _f;
        uint256 _g;
        uint256 _h;
        assembly ("memory-safe") {

            function f_a(a) -> b {
                let a1 := sload(sub(a, 1))
                let x := mload(a1)
                let b2 := sload(add(a, 1))
                b := sload(b2)
                b := add(b, sload(x))
                b := add(b, add(x, b))
            }

            _a := f_a(sload(0))
            _b := add(f_a(1), f_a(2))
            _c := add(f_a(3), f_a(2))
            _d := add(f_a(4), f_a(3))

            _e := add(f_a(5), f_a(4))
            _f := add(f_a(6), f_a(5))
            _g := add(f_a(7), f_a(6))
            _h := add(f_a(8), f_a(7))
            mstore(0x40, 0x80)
        }
        uint256[2] memory sum;
        sum[0] = _a + _b + _c + _d + _e + _f + _g + _h;
        return sum[0];
    }

}
```

**Optimized code snippet storing values in memory :**

```jsx
let _2 := sload(/** @src 0:92:1365  "contract Exp5 {..." */ _1)
/// @src 0:602:1247  "assembly (\"memory-safe\") {..."
mstore(0xc0, mload(sload(add(_2, not(0)))))
let usr$b := sload(sload(add(_2, 1)))
mstore(0xa0, add(usr$b, sload(mload(0xc0))))
mstore(0x80, mload(sload(1)))
let _3 := sload(3)
let usr$b_1 := sload(_3)
let usr$b_2 := add(usr$b_1, sload(mload(0x80)))
mstore(0x01a0, mload(_2))
let _4 := sload(2)
let usr$b_3 := sload(_4)
let usr$b_4 := add(usr$b_3, sload(mload(0x01a0)))
let usr$x := mload(_4)
let _5 := sload(/** @src 0:92:1365  "contract Exp5 {..." */ 4)
/// @src 0:602:1247  "assembly (\"memory-safe\") {..."
let usr$b_5 := sload(_5)
mstore(0x0140, add(usr$b_5, sload(usr$x)))
```

After this step the free memory pointer is updated by the optimizer and we again updating it in our code

```jsx
     mstore(0x40, 0x80) // This will update the fmp
        }
        uint256[2] memory sum; // This will override the values stored in the fmp and fmp + 0x20 as it is 2 length array
//Returns different Value
```

This is achieved by manipulating the stack size during optimization, triggering the StackLimitEvader to overwrite values in memory, leading to a different result when optimized.

**Output**

![Output](https://file.notion.so/f/s/6e88223a-9cd3-4a44-875b-0f82e16a8ee2/photo_6300695497312154375_x.jpg?id=53a09889-48dd-4b27-acdb-b24fd1b56773&table=block&spaceId=ab48cbd9-e93c-449f-912e-ad13ad550a0e&expirationTimestamp=1690596000000&signature=GkjC84J2qf4K6eQ2S2hFnxSfaivzC8FHIl2rZbwja3Y&downloadName=photo_6300695497312154375_x.jpg)

## What I learned

Through these tasks, I have not only learned the YUL language but also delved deep into the internals of YUL Optimization and memory-safety. This exploration helped me to gain a better understanding of the EVM .

## Sources

-   [Yul Docs](https://docs.soliditylang.org/en/latest/yul.html#memoryguard)
-   [Solidity_Blog](https://blog.soliditylang.org/2021/11/09/solidity-0.8.10-release-announcement/)
-   [StackLimitEvader](https://github.com/ethereum/solidity/blob/29751849b58a7db5eec3f297087544ae9531f0df/libyul/optimiser/StackLimitEvader.cpp)
-   [FullInliner](https://docs.soliditylang.org/en/latest/internals/optimizer.html#fullinliner).
