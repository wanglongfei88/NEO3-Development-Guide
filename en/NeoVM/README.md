﻿# Neo Virtual Machine

<!-- TOC -->

- [Neo Virtual Machine](#neo-virtual-machine)
    - [Changes in NEO3](#changes-in-neo3)
    - [NeoVM Architecture](#neovm-architecture)
        - [Execution Engine](#execution-engine)
        - [Temporary Storage](#temporary-storage)
    - [Interoperable service layer](#interoperable-service-layer)
    - [Built-in data types](#built-in-data-types)
    - [Instructions](#instructions)
    - [Fee](#fee)

<!-- /TOC -->


NeoVM is a lightweighted, general-purpose virtual machine that executes Neo smart contract code. The concept of virtual machine described in this paadper is in narrow sense, it's not a simulation of physical machine by operating system. Unlike VMware or Hyper-V, it's mainly aimed at specific usage.

For example, in JVM or CLR of .Net, source code will be compiled into relevant bytecodes, and be executed on the corresponding virtual machine. JVM or CLR will read instructions, decode, execute and write results back. Those steps are very similar to the concepts on real physical machines. The binary instructions are still running on the physical machine. It takes instructions from memory and transmits them to the CPU through the bus, then decodes, executes and stores the results.

## Changes in NEO3

- ADD
    -  OpCode: [DUPFROMALTSTACKBOTTOM](#stack-operation)

- DELETE
    - `APPCALL`, `TAILCALL`, `SHA1`, `SHA256`, `HASH160`, `HASH256`, `CHECKSIG`, `VERIFY`, `CHECKMULTISIG`, `CALL_I`, `CALL_E`, `CALL_ED`, `CALL_ET`, `CALL_EDT`

## NeoVM Architecture
![nvm](../../images/nvm.jpg)

The graph above is the system architecture of NeoVM, which includes execution engine, memory, interoperable services.

A complete operation process is as follows:

1. Compile the smart contract source codes into bytecodes.

2. Push the bytecodes and related parameters as a running context into the `InvocationStack`.

3. Each time the execution engine takes a instruction from the current context, it executes the instruction, and stores the data in the evaluation stack (`EvaluationStack`) and temporary stack (`AltStack`) of current context.

4. It uses the interoperable service if it needs to access external data.

5. After all scripts are executed, the results will be saved in the `ResultStack`.

### Execution Engine

The left part of the graph above is the virtual machine's execution engine(equivalent to CPU). It can execute common instructions such as flow control, stack operation, bit operation, arithmetic operation, logical operation, cryptography, etc. It can also interact with the interoperable services through system call. NeoVM has four states: `NONE`, `HALT`, `FAULT`, `BREAK`.

- `NONE` is a normal state.

- `HALT` is a stopped state. When the `InvocationStack` is empty, it means all scripts are executed, the virtual machine's state will be set to `HALT`.

- `FAULT` is an error state. When something goes wrong with an operation, the virtual machine's state will be set to `FAULT`.

- `BREAK` is an interrupted state and is used in the debugging process of smart contract generally.

Each time before the virtual machine starts, the execution engine will detect the virtual machine's state, and only when the state is `NONE`, can it start running.


### Temporary Storage

NeoVM uses stacks as its temporary storage. NeoVM has four types of stacks: `InvocationStack`, `EvaluationStack`, `AltStack` and `ResultStack`.

- `InvocationStack`: *is used to store all execution contexts of current NeoVM, which are isolated from each other in the stack. Context switching is performed based on the current context and entry context. The current context points to the top element of invocation stack, which is ExecutionContext0 in the architecture figure. And the entry context points to the tail element of the invocation stack, which is ExecutionContextN in the architecture figure*
- *`EvaluationStack` is for storing the data used by the instruction in the execution process. Each execution context has its evaluation stack*
- *`AltStack` is for storing the temporary data used by the instruction in the execution process. Each execution context has its alt stack*
- *`ResultStack` is used to store execution results after all scripts are executed.*


## Interoperable service layer

The right part of the graph above is the virtual machine's interoperable service layer, which provides API for smart contract to access data on the blockchain. By using these API, smart contract can access information in a block , information in a transaction, and information of an asset.

In addition, the interoperable service layer provides a persistent storage for each contract. Each Neo contract can declare to have a private storage to save information in form of key-value pair optionally when the contract is created. When a smart contract invocate another smart contract, the storage are separated. The smart contract being invoked has it's own persistent storage. If the caller contract needs the contract being invoked to access data of the caller contract, it needs to pass its own storage context to the callee (authorization) before the callee can perform the read-write operation.

The detail of interoperable services are describled in the "Smart Contract" section.

## Built-in data types

NeoVM has seven built-in data types:

| Type | Description |
|------|------|
| Boolean |  Implemented as two byte arrays, `TRUE` and `FALSE`.  |
| Integer | Implemented as a `BigInteger` value.  |
| ByteArray |Implemented as a byte array.  |
| Array |  Implemented as a `List<StackItem>`, the `StackItem` is an abstract class, and all the built-in data types are inherited from it. |
| Struct |  Inherited from Array, a `Clone` method is added and `Equals` method is overridden. |
| Map | Implemented as a key-value pair `Dictionary<StackItem, StackItem>`.  |
| InteropInterface |  Interoperable interface |


```c#
// boolean type
private static readonly byte[] TRUE = { 1 };
private static readonly byte[] FALSE = new byte[0];

private bool value;
```


## Instructions

NeoVM has implemented 173 instructions. The categories are as follows:

| Contrant | Flow Control | Stack Operation | String Operation | Logical Operation | Arithmetic Operation | Advanced Data Structure | Exception Processing |
| ---- | -------- | ------ | ------ | -------- | -------- | -------- | ---- |
| 98 | 7 | 17 | 5 | 5 | 25 | 14 | 2 |


### Contrant

The constant instructions mainly complete the function of pushing constants or arrays into the `EvaluationStack`.

#### PUSH0

| Instruction   | PUSH0                                 |
|--------|----------|
| Bytecode: | 0x00                                  |
| Alias: |   `PUSHF`                |
| Fee: | 3e-7 GAS                           |
| Function: | Push an empty array into the `EvaluationStack`  |

#### PUSHBYTES

| Instruction   | PUSHBYTES1\~PUSHBYTES75                                    |
|----------|-----------------------------|
| Bytecode: | 0x01\~0x4B                                                 |
| Fee: | 1.2e-6 GAS                           |
| Function:   | Push a byte array into the `EvaluationStack`, the length of which is equal to the value of this instruction's bytecode. |

#### PUSHDATA

| Instruction   | PUSHDATA1, PUSHDATA2, PUSHDATA4                                   |
|----------|---------------------------------------|
| Bytecode: | 0x4C, 0x4D, 0x4E                                                  |
| Fee: | 1.8e-6 GAS, 1.3e-4 GAS, 1.1e-3 GAS                    |
| Function:   | Push a byte array into the `EvaluationStack`, the length of which is specified by 1\|2\|4 bytes after this instruction.  |

#### PUSHM1

| Instruction   | PUSHM1                                   |
|----------|------------------------------------------|
| Bytecode: | 0x4F                                     |
| Fee: | 3e-7 GAS                             |
| Function:   | Push a BigInteger of `-1`  into the `EvaluationStack`. |

#### PUSHN

| Instruction   | PUSH1\~PUSH16                               |
|----------|---------------------------------------------|
| Bytecode: | 0x51\~0x60                                  |
| Alias:   |  `PUSHT` is an alias for `PUSH1`      |
| Fee: | 3e-7 GAS                                      |
| Function:   | Push a BigInteger into the `EvaluationStack`, the value of which is equal to 1\~16. |

### Flow Control

It's used to control the running process of NeoVM, including jump, call and other instructions.

#### NOP

| Instruction   | NOP                                         |
|----------|---------------------------------------------|
| Bytecode: | 0x61                                        |
| Fee: | 3e-7 GAS                                |
| Function:   | Empty operation, but will add 1 to the instruction counter. |

#### JMP

| Instruction   | JMP                                                     |
|----------|---------------------------------------------------------|
| Bytecode: | 0x62                                                    |
| Fee: | 7e-7 GAS                                            |
| Function:   | Jump to the specified offset unconditionally, which is specified by 2 bytes after this instruction. |

#### JMPIF

| Instruction   | JMPIF                                                                                                                |
|----------|-------------------------------------------------------------|
| Bytecode: | 0x63      |
| Fee: | 7e-7 GAS                                                                                                         |
| Function:   | When the top element of the `EvaluationStack` isn't 0, then jump to the specified offset, which is specified by 2 bytes after this instruction. </br> Whether the condition determines true or not, the top element of the stack will be removed.  |

#### JMPIFNOT

| Instruction   | JMPIFNOT                                                           |
|----------|--------------------------------------------------------------------|
| Bytecode: | 0x64                                                               |
| Fee: | 7e-7 GAS                                                        |
| Function:   | When the top element of the `EvaluationStack` is 0, then jump to the specified offset, which is specified by 2 bytes after this instruction. |

#### CALL

| Instruction   | CALL                                                  |
|----------|-------------------------------------------------------|
| Bytecode: | 0x65                                                  |
| Fee: | 2.2e-4 GAS                           |
| Function:   | Call the function at the specified offset, which is specified by 2 bytes after this instruction.  |

#### RET

| Instruction   | RET                                                                                              |
|----------|--------------------------------------------------------------------------------------------------|
| Bytecode: | 0x66                                                                                             |
| Fee: | 4e-7 GAS                                                        |
| Function:   | Remove the top element of the `InvocationStack` and set the instruction counter point to the next frame of the stack. </br> If the `InvocationStack` is empty, the virtual machine enters `HALT` state.  |

#### SYSCALL

| Instruction   | SYSCALL                                                |
|----------|--------------------------------------------------------|
| Bytecode: | 0x68                                                   |
| Fee: | 0 GAS                                                        |
| Function:   | Call the specified interoperable function whose name is specified by the string after this instruction. |

### Stack Operation

Copy, remove and swap the elements of the stack.

#### DUPFROMALTSTACKBOTTOM 
>  **added in NEO3**

| Instruction   | DUPFROMALTSTACKBOTTOM            |
|--------|------------------------------------------|
| Bytecode: | 0x69                                     |
| Fee: | 6e-7 GAS                                                        |
| Function:   | Copy the bottom element of the `AltStack`, and push it into the `EvaluationStack `. |

#### DUPFROMALTSTACK

| Instruction   | DUPFROMALTSTACK                          |
|--------|------------------------------------------|
| Bytecode: | 0x6A                                     |
| Fee: | 6e-7 GAS                                                        |
| Function:   | Copy the top element of the `AltStack`, and push it into the `EvaluationStack `. |

#### TOALTSTACK

| Instruction   | TOALTSTACK                               |
|----------|------------------------------------------|
| Bytecode: | 0x6B                                     |
| Fee: | 6e-7 GAS                                                        |
| Function:   | Remove the top element of the `EvaluationStack`, and push it into the `AltStack`. |

#### FROMALTSTACK

| Instruction   | FROMALTSTACK                             |
|----------|------------------------------------------|
| Bytecode: | 0x6C                                     |
| Fee: | 6e-7 GAS                                                        |
| Function:   | Remove the top element of the `AltStack`, and push it into the `EvaluationStack`. |

#### XDROP

| Instruction   | XDROP                                              |
|----------|----------------------------------------------------|
| Bytecode: | 0x6D                                               |
| Fee: | 4e-6 GAS                                                        |
| Function:   | Remove the element n at the top of the `EvaluationStack`, and remove the remaining element with index n. |
| Input:   | Xn Xn-1 ... X2 X1 X0 n                             |
| Output:   | Xn-1 ... X2 X1 X0                                  |

#### XSWAP

| Instruction   | XSWAP                                                                   |
|----------|-------------------------------------------------------------------------|
| Bytecode: | 0x72                                                                    |
| Fee: | 6e-7 GAS                                                        |
| Function:   | Remove the element n at the top of the `EvaluationStack`, and swap the remaining element with index 0 and the element with index n. |
| Input:   | Xn Xn-1 ... X2 X1 X0 n                                                  |
| Output:   | X0 Xn-1 ... X2 X1 Xn                                                    |

#### XTUCK

| Instruction   | XTUCK                                                                     |
|----------|---------------------------------------------------------------------------|
| Bytecode: | 0x73                                                                      |
| Fee: | 4e-6 GAS                                                        |
| Function:   |  Remove the element n at the top of the `EvaluationStack`, copy the element with index 0, and insert to the index n.  |
| Input:   | Xn Xn-1 ... X2 X1 X0 n                                                    |
| Output:   | Xn X0 Xn-1 ... X2 X1 X0                                                   |

#### DEPTH

| Instruction   | DEPTH                                  |
|----------|----------------------------------------|
| Bytecode: | 0x74                                   |
| Fee: | 6e-7 GAS                                                        |
| Function:   | Push the numbe of elements in the `EvaluationStack` into the top of the `EvaluationStack`. |

#### DROP

| Instruction   | DROP                   |
|----------|------------------------|
| Bytecode: | 0x75                   |
| Fee: | 6e-7 GAS                                                        |
| Function:   | Remove the top element of the `EvaluationStack` |

#### DUP

| Instruction   | DUP                    |
|----------|------------------------|
| Bytecode: | 0x76                   |
| Fee: | 6e-7 GAS                                                        |
| Function:   | Copy the top element of the `EvaluationStack`, and push it into the `EvaluationStack`. |
| Input:   | X                      |
| Output:   | X X                    |

#### NIP

| Instruction   | NIP                         |
|----------|-----------------------------|
| Bytecode: | 0x77                        |
| Fee: | 6e-7 GAS                                                        |
| Function:   | Remove the second top element of the `EvaluationStack` |
| Input:   | X1 X0                       |
| Output:   | X0                          |

#### OVER 

| Instruction   | OVER                                     |
|----------|------------------------------------------|
| Bytecode: | 0x78                                     |
| Fee: | 6e-7 GAS                                                        |
| Function:   |  Copy the second top element of the `EvaluationStack`, and push it into the `EvaluationStack`. |
| Input:   | X1 X0                                    |
| Output:   | X1 X0 X1                                 |

#### PICK 

| Instruction   | PICK                                                       |
|----------|------------------------------------------------------------|
| Bytecode: | 0x79                                                       |
| Fee: | 6e-7 GAS                                                        |
| Function:   | Remove the element n at the top of the `EvaluationStack`, and copy the element with index n to the top. |
| Input:   | Xn Xn-1 ... X2 X1 X0 n                                     |
| Output:   | Xn Xn-1 ... X2 X1 X0 Xn                                    |

#### ROLL 

| Instruction   | ROLL                                                       |
|----------|------------------------------------------------------------|
| Bytecode: | 0x7A                                                       |
| Fee: | 4e-6 GAS                                                        |
| Function:   | Remove the element n at the top of the `EvaluationStack`, and move the element with index n to the top.  |
| Input:   | Xn Xn-1 ... X2 X1 X0 n                                     |
| Output:   | Xn-1 ... X2 X1 X0 Xn                                       |

#### ROT 

| Instruction   | ROT                                         |
|----------|---------------------------------------------|
| Bytecode: | 0x7B                                        |
| Fee: | 6e-7 GAS                                                        |
| Function:   | Move the third top element of the `EvaluationStack` to the top.  |
| Input:   | X2 X1 X0                                    |
| Output:   | X1 X0 X2                                    |

#### SWAP 

| Instruction   | SWAP                           |
|----------|--------------------------------|
| Bytecode: | 0x7C                           |
| Fee: | 6e-7 GAS                                                        |
| Function:   | Swap the two elements at the top of the `EvaluationStack` |
| Input:   | X1 X0                          |
| Output:   | X0 X1                          |

#### TUCK 

| Instruction   | TUCK                                  |
|----------|---------------------------------------|
| Bytecode: | 0x7D                                  |
| Fee: | 6e-7 GAS                                                        |
| Function:   | Copy the top element of the `EvaluationStack`, and insert to the index 2. |
| Input:   | X1 X0                                 |
| Output:   | X0 X1 X0                              |


### String Operation

#### CAT

| Instruction   | CAT                                              |
|----------|--------------------------------------------------|
| Bytecode: | 0x7E                                             |
| Fee: | 8e-4 GAS                                                        |
| Function:   | Remove the two top elements of the `EvaluationStack`, concat them together and push it back to the `EvaluationStack` |
| Input:   | X1 X0                                            |
| Output:   | Concat(X1,X0)                                    |

#### SUBSTR

| Instruction   | SUBSTR                                       |
|----------|----------------------------------------------|
| Bytecode: | 0x7F                                         |
| Fee: | 8e-4 GAS                                                        |
| Function:   | Remove the three top elements of the `EvaluationStack`, calculate the substring and push it back. |
| Input:   | X index len                                  |
| Output:   | SubString(X,index,len)                       |

#### LEFT

| Instruction   | LEFT                                         |
|----------|----------------------------------------------|
| Bytecode: | 0x80                                         |
| Fee: | 8e-4 GAS                                                        |
| Function:   | Remove the two top elements of the `EvaluationStack`, calculate the left-side substring and push it back. |
| Input:   | X len                                        |
| Output:   | Left(X,len)                                  |

#### RIGHT

| Instruction   | RIGHT                                        |
|----------|----------------------------------------------|
| Bytecode: | 0x81                                         |
| Fee: | 8e-4 GAS                                                        |
| Function:   | Remove the two top elements of the `EvaluationStack`, calculate the right-side substring and push it back. |
| Input:   | X len                                        |
| Output:   | Right(X,len)                                 |

#### SIZE

| Instruction   | SIZE                             |
|----------|----------------------------------|
| Bytecode: | 0x82                             |
| Fee: | 6e-7 GAS                                                        |
| Function:   | Push the length of the top string element to the `EvaluationStack` top.  |
| Input:   | X                                |
| Output:   | X len(X)                         |


### Logical Operation

#### INVERT

| Instruction   | INVERT                       |
|----------|------------------------------|
| Bytecode: | 0x83                         |
| Fee: | 1e-6 GAS                                                        |
| Function:   | Remove the top element, inverse by bit, and push it back to the `EvaluationStack` top.  |
| Input:   | X                            |
| Output:   | \~X                          |

#### AND

| Instruction   | AND                                    |
|----------|----------------------------------------|
| Bytecode: | 0x84                                   |
| Fee: | 2e-6 GAS                                                        |
| Function:   | Remove the two top elements, push the logic AND result of the two elements back to the `EvaluationStack` top. |
| Input:   | AB                                     |
| Output:   | A&B                                    |

#### OR

| Instruction   | OR                                     |
|----------|----------------------------------------|
| Bytecode: | 0x85                                   |
| Fee: | 2e-6 GAS                                                        |
| Function:   | Remove the two top elements, push the logic OR result of the two elements back to the `EvaluationStack` top.  |
| Input:   | AB                                     |
| Output:   | A\|B                                   |

#### XOR

| Instruction   | XOR                                      |
|----------|------------------------------------------|
| Bytecode: | 0x86                                     |
| Fee: | 2e-6 GAS                                                        |
| Function:   | Remove the two top elements, push the logic XOR result of the two elements back to the `EvaluationStack` top.  |
| Input:   | AB                                       |
| Output:   | A\^B                                     |

#### EQUAL

| Instruction   | EQUAL                                        |
|----------|----------------------------------------------|
| Bytecode: | 0x87                                         |
| Fee: | 2e-6 GAS                                                        |
| Function:   | Check whether the top two elements are equivalence bit-by-bit. |
| Input:   | AB                                           |
| Output:   | Equals(A,B)                                  |

### Arithmetic Operation

#### INC

| Instruction   | INC                                |
|----------|------------------------------------|
| Bytecode: | 0x8B                               |
| Fee: | 1e-6 GAS                                                        |
| Function:   | Add 1 to the top element of the `EvaluationStack`.   |
| Input:   | X                                  |
| Output:   | X+1                                |

#### DEC

| Instruction   | DEC                                |
|----------|------------------------------------|
| Bytecode: | 0x8C                               |
| Fee: | 1e-6 GAS                                                        |
| Function:   | Add -1 to the top element of the `EvaluationStack`. |
| Input:   | X                                  |
| Output:   | X-1                                |

#### SIGN

| Instruction   | SIGN                                         |
|----------|----------------------------------------------|
| Bytecode: | 0x8D                                         |
| Fee: | 1e-6 GAS                                                        |
| Function:   | Remove the top element and push the sign of it back to the `EvaluationStack`. |
| Input:   | X                                            |
| Output:   | X.Sign()                                     |

#### NEGATE

| Instruction   | NEGATE                         |
|----------|--------------------------------|
| Bytecode: | 0x8F                           |
| Fee: | 1e-6 GAS                                                        |
| Function:   | Remove the top element and push the opposite number back to the `EvaluationStack`.  |
| Input:   | X                              |
| Output:   | \-X                            |

#### ABS

| Instruction   | ABS                            |
|----------|--------------------------------|
| Bytecode: | 0x90                           |
| Fee: | 1e-6 GAS                                                        |
| Function:   | Remove the top element and push the absolute number back to the `EvaluationStack`.  |
| Input:   | X                              |
| Output:   | Abs(X)                         |

#### NOT

| Instruction   | NOT                                |
|----------|------------------------------------|
| Bytecode: | 0x91                               |
| Fee: | 1e-6 GAS                                                        |
| Function:   | Remove the top element and push the logic "negation" value back to the `EvaluationStack`.  |
| Input:   | X                                  |
| Output:   | !X                                 |

#### NZ

| Instruction   | NZ                                  |
|----------|-------------------------------------|
| Bytecode: | 0x92                                |
| Fee: | 1e-6 GAS                                                        |
| Function:   | Check whether the top element of the `EvaluationStack` is a non-zero value. |
| Input:   | X                                   |
| Output:   | X!=0                                |

#### ADD

| Instruction   | ADD                                    |
|----------|----------------------------------------|
| Bytecode: | 0x93                                   |
| Fee: | 2e-6 GAS                                                        |
| Function:   | The addition operation is performed on the top two elments of the `EvaluationStack`.  |
| Input:   | AB                                     |
| Output:   | A+B                                    |

#### SUB

| Instruction   | SUB                                    |
|----------|----------------------------------------|
| Bytecode: | 0x94                                   |
| Fee: | 2e-6 GAS                                                        |
| Function:   | The subtraction operation is performed on the top two elments of the `EvaluationStack`.    |
| Input:   | AB                                     |
| Output:   | A-B                                    |

#### MUL

| Instruction   | MUL                                    |
|----------|----------------------------------------|
| Bytecode: | 0x95                                   |
| Fee: | 3e-6 GAS                                                        |
| Function:   | The multiplication operation is performed on the top two elments of the `EvaluationStack`.  |
| Input:   | AB                                     |
| Output:   | A\*B                                   |

#### DIV

| Instruction   | DIV                                    |
|----------|----------------------------------------|
| Bytecode: | 0x96                                   |
| Fee: | 3e-6 GAS                                                        |
| Function:  | The division operation is performed on the top two elments of the `EvaluationStack`.   |
| Input:   | AB                                     |
| Output:   | A/B                                    |

#### MOD

| Instruction   | MOD                                    |
|----------|----------------------------------------|
| Bytecode: | 0x97                                   |
| Fee: | 3e-6 GAS                                                        |
| Function:   | The redundancy operation is performed on the top two elments of the `EvaluationStack`.   |
| Input:   | AB                                     |
| Output:   | A%B                                    |

#### SHL

| Instruction   | SHL                              |
|----------|----------------------------------|
| Bytecode: | 0x98                             |
| Fee: | 3e-6 GAS                                                        |
| Function:   | The left-shift operation is performed on the top elment of the `EvaluationStack`.  |
| Instruction   | Xn                               |
| Bytecode: | X\<\<n                           |

#### SHR

| Instruction   | SHR                              |
|----------|----------------------------------|
| Bytecode: | 0x99                             |
| Fee: | 3e-6 GAS                                                        |
| Function:   | The right-shift operation is performed on the top elment of the `EvaluationStack`.  |
| Input:   | Xn                               |
| Output:   | X\>\>n                           |

#### BOOLAND

| Instruction   | BOOLAND                                |
|----------|----------------------------------------|
| Bytecode: | 0x9A                                   |
| Fee: | 2e-6 GAS                                                        |
| Function:   | The logic "and" operation is performed on the top two elments of the `EvaluationStack`. |
| Input:   | AB                                     |
| Output:   | A&&B                                   |

#### BOOLOR

| Instruction   | BOOLOR                                 |
|----------|----------------------------------------|
| Bytecode: | 0x9D                                   |
| Fee: | 2e-6 GAS                                                        |
| Function:   | The logic "or" operation is performed on the top two elments of the `EvaluationStack`.  |
| Input:   | AB                                     |
| Output:   | A\|\|B                                 |

#### NUMEQUAL

| Instruction   | NUMEQUAL                               |
|----------|----------------------------------------|
| Bytecode: | 0x9C                                   |
| Fee: | 2e-6 GAS                                                        |
| Function:   | Check whether the top two Bitintegers of the `EvaluationStack` are equal.   |
| Input:   | AB                                     |
| Output:   | A==B                                   |

#### NUMNOTEQUAL

| Instruction   | NUMNOTEQUAL                              |
|----------|------------------------------------------|
| Bytecode: | 0x9E                                     |
| Fee: | 2e-6 GAS                                                        |
| Function:   | Check whether the top two Bitintegers of the `EvaluationStack` aren't equal.  |
| Input:   | AB                                       |
| Output:   | A!=B                                     |

#### LT 

| Instruction   | LT                                     |
|----------|----------------------------------------|
| Bytecode: | 0x9F                                   |
| Fee: | 2e-6 GAS                                                        |
| Function:   | Check whether the first top element is less than the second top element in the `EvaluationStack`.  |
| Input:   | AB                                     |
| Output:   | A\<B                                   |

#### GT

| Instruction   | GT                                     |
|----------|----------------------------------------|
| Bytecode: | 0xA0                                   |
| Fee: | 2e-6 GAS                                                        |
| Function:   | Check whether the first top element is more than the second top element in the `EvaluationStack`.   |
| Input:   | AB                                     |
| Output:   | A\>B                                   |

#### LTE

| Instruction   | LTE                                        |
|----------|--------------------------------------------|
| Bytecode: | 0xA1                                       |
| Fee: | 2e-6 GAS                                                        |
| Function:   | Check whether the first top element ls less than or equal to the second top element in the `EvaluationStack`.  |
| Input:   | AB                                         |
| Output:   | A\<=B                                      |

#### GTE

| Instruction   | GTE                                        |
|----------|--------------------------------------------|
| Bytecode: | 0xA2                                       |
| Fee: | 2e-6 GAS                                                        |
| Function:   | Check whether the first top element is more than or equal to the second top element in the `EvaluationStack`. |
| Input:   | AB                                         |
| Output:   | A\>=B                                      |

#### MIN

| Instruction   | MIN                                    |
|----------|----------------------------------------|
| Bytecode: | 0xA3                                   |
| Fee: | 2e-6 GAS                                                        |
| Function:   | Calculate the minimum of the two top elements in the `EvaluationStack`.  |
| Input:   | AB                                     |
| Output:   | Min(A,B)                               |

#### MAX

| Instruction   | MAX                                    |
|----------|----------------------------------------|
| Bytecode: | 0xA4                                   |
| Fee: | 2e-6 GAS                                                        |
| Function:   | Calculate the maximum of the two top elements in the `EvaluationStack`. |
| Input:   | AB                                     |
| Output:   | Max(A,B)                               |

#### WITHIN

| Instruction   | WITHIN                                       |
|----------|----------------------------------------------|
| Bytecode: | 0xA5                                         |
| Fee: | 2e-6 GAS                                                        |
| Function:   | Check whether the Biginteger value is within the specified range.  |
| Input:   | XAB                                          |
| Output:   | A\<=X&&X\<B                                  |

### Advanced Data Structure

It has implemented common operations for array, map, struct, etc.

### ARRAYSIZE

| Instruction   | ARRAYSIZE                        |
|----------|----------------------------------|
| Bytecode: | 0xC0                             |
| Fee: | 1.5e-6 GAS                                                        |
| Function:   | Get the number of elements of the array at the top of the `EvaluationStack`. |
| Input:   | [X0 X1 X2 ... Xn-1]              |
| Output:   | n                                |

#### PACK

| Instruction   | PACK                              |
|----------|-----------------------------------|
| Bytecode: | 0xC1                              |
| Fee: | 7e-5 GAS                                                        |
| Function:   | Pack the n elments at the top of the `EvaluationStack` into array. |
| Input:   | Xn-1 ... X2 X1 X0 n               |
| Output:   | [X0 X1 X2 ... Xn-1]               |

#### UNPACK

| Instruction   | UNPACK                             |
|----------|------------------------------------|
| Bytecode: | 0xC2                               |
| Fee: | 7e-5 GAS                                                        |
| Function:   | Get the number of elements of the array at the top of the `EvaluationStack`.  |
| Input:   | [X0 X1 X2 ... Xn-1]                |
| Output:   | Xn-1 ... X2 X1 X0 n                |

#### PICKITEM

| Instruction   | PICKITEM                           |
|----------|------------------------------------|
| Bytecode: | 0xC3                               |
| Fee: | 2.7e-3 GAS                                                        |
| Function:   | Get the specified element in the array at the top of the `EvaluationStack`. |
| Input:   | [X0 X1 X2 ... Xn-1] i              |
| Output:   | Xi                                 |

#### SETITEM\*

| Instruction   | SETITEM                                  |
|----------|------------------------------------------|
| Bytecode: | 0xC4                                     |
| Fee: | 2.7e-3 GAS                                                        |
| Function:   | Assign a value to the specified index element in the array at the top of the `EvaluationStack`. |
| Input:   | [X0 X1 X2 ... Xn-1] I V                  |
| Output:   | [X0 X1 X2 Xi-1 V X i+1 ... Xn-1]         |

#### NEWARRAY

| Instruction   | NEWARRAY                           |
|----------|------------------------------------|
| Bytecode: | 0xC5                               |
| Fee: | 1.5e-4 GAS                                                        |
| Function:   | Create a new N-size array on the top of the `EvaluationStack`. |
| Input:   | n                                  |
| Output:   | Array(n) with all `false` elements.         |

#### NEWSTRUCT

| Instruction   | NEWSTRUCT                           |
|----------|-------------------------------------|
| Bytecode: | 0xC6                                |
| Fee: | 1.5e-4 GAS                                                        |
| Function:   | Create a new N-size struct on the top of the `EvaluationStack`. |
| Input:   | n                                   |
| Output:   | Struct(n) with all `false` elements.        |

#### NEWMAP

| Instruction   | NEWMAP                  |
|----------|-------------------------|
| Bytecode: | 0xC7                    |
| Fee: | 2e-6 GAS                                                        |
| Function:   | Create a new map on the top of the `EvaluationStack`.  |
| Input:   |                       |
| Output:   | Map()                   |

#### APPEND*

| Instruction   | APPEND                |
|----------|-----------------------|
| Bytecode: | 0xC8                  |
| Fee: | 1.5e-4 GAS                                                        |
| Function:   | Add a new item to the array |
| Input:   | Array item            |
| Output:   | Array.add(item)       |

#### REVERSE*

| Instruction   | REVERSE             |
|----------|---------------------|
| Bytecode: | 0xC9                |
| Fee: | 5e-6 GAS                                                        |
| Function:   | Reverse the array. |
| Input:   | [X0 X1 X2 ... Xn-1] |
| Output:   | [Xn-1 ... X2 X1 X0] |

#### REMOVE*

| Instruction   | REMOVE                            |
|----------|-----------------------------------|
| Bytecode: | 0xCA                              |
| Fee: | 5e-6 GAS                                                        |
| Function:   | Remove the specified element from the array or map.     |
| Input:   | [X0 X1 X2 ... Xn-1] m             |
| Output:   | [X0 X1 X2 ... Xm-1 Xm+1 ... Xn-1] |

#### HASKEY

| Instruction   | HASKEY                              |
|----------|-------------------------------------|
| Bytecode: | 0xCB                                |
| Fee: | 2.7e-3 GAS                                                        |
| Function:   |  Check whether the array or the map contains a specified key element. |
| Input:   | [X0 X1 X2 ... Xn-1] key             |
| Output:   | true or false                       |

#### KEYS

| Instruction   | KEYS                                |
|----------|-------------------------------------|
| Bytecode: | 0xCC                                |
| Fee: | 5e-6 GAS                                                        |
| Function:   | Get all the keys of the map, and put them into a new array. |
| Input:   | Map                                 |
| Output:   | [key1 key2 ... key n]               |

#### VALUES

| Instruction   | VALUES                                  |
|----------|-----------------------------------------|
| Bytecode: | 0xCD                                    |
| Fee: | 7e-5 GAS                                                        |
| Function:   | Get all the values of the array or the map, and put them into a new array. |
| Input:   | Map or Array                              |
| Output:   | [Value1 Value2... Value n]              |

### Exception Processing

#### THROW

| Instruction   | THROW                 |
|----------|-----------------------|
| Bytecode: | 0xF0                  |
| Fee: | 3e-7 GAS                                                        |
| Function:   | Set the virtual machine state to `FAULT` |

#### THROWIFNOT

| Instruction   | THROWIFNOT                                                       |
|----------|------------------------------------------------------------------|
| Bytecode: | 0xF1                                                             |
| Fee: | 3e-7 GAS                                                        |
| Function:   | Read a boolean value from the top of the stack, and if it's False, then set the virtual machine state to `FAULT`. |

Note: The operation code with \* indicates that the result of the operation is not pushed back to the `EvaluationStack`.

## Fee

| OpCode | Fee (GAS) |
|---|---|
| PUSH0 | 0.00000030 |
| PUSHBYTES1 ~ PUSHBYTES75 | 0.00000120 |
| PUSHDATA1 | 0.00000180 |
| PUSHDATA2 | 0.00013000 |
| PUSHDATA4 | 0.00110000 |
| PUSHM1 | 0.00000030 |
| PUSH1 ~ PUSH16 | 0.00000030 |
| NOP | 0.00000030 |
| JMP | 0.00000070 |
| JMPIF | 0.00000070 |
| JMPIFNOT | 0.00000070 |
| CALL | 0.00022000 |
| RET | 0.00000040 |
| SYSCALL | 0 |
| DUPFROMALTSTACKBOTTOM | 0.00000060 |
| DUPFROMALTSTACK | 0.00000060 |
| TOALTSTACK | 0.00000060 |
| FROMALTSTACK | 0.00000060 |
| XDROP | 0.00000400 |
| XSWAP | 0.0000006 |
| XTUCK | 0.000004 |
| DEPTH | 0.0000006 |
| DROP     | 0.0000006 |
| DUP     | 0.0000006 |
| NIP     | 0.0000006 |
| OVER     | 0.0000006 |
| PICK     | 0.0000006 |
| ROLL     | 0.000004 |
| ROT     | 0.0000006 |
| SWAP     | 0.0000006 |
| TUCK     | 0.0000006 |
| CAT     | 0.0008 |
| SUBSTR     | 0.0008 |
| LEFT     | 0.0008 |
| RIGHT     | 0.0008 |
| SIZE     | 0.0000006 |
| INVERT     | 0.000001 |
| AND     | 0.000002 |
| OR     | 0.000002 |
| XOR     | 0.000002 |
| EQUAL     | 0.000002 |
| INC     | 0.000001 |
| DEC     | 0.000001 |
| SIGN     | 0.000001 |
| NEGATE     | 0.000001 |
| ABS     | 0.000001 |
| NOT     | 0.000001 |
| NZ     | 0.000001 |
| ADD     | 0.000002 |
| SUB     | 0.000002 |
| MUL     | 0.000003 |
| DIV     | 0.000003 |
| MOD     | 0.000003 |
| SHL     | 0.000003 |
| SHR     | 0.000003 |
| BOOLAND     | 0.000002 |
| BOOLOR     | 0.000002 |
| NUMEQUAL     | 0.000002 |
| NUMNOTEQUAL     | 0.000002 |
| LT     | 0.000002 |
| GT     | 0.000002 |
| LTE     | 0.000002 |
| GTE     | 0.000002 |
| MIN     | 0.000002 |
| MAX     | 0.000002 |
| WITHIN     | 0.000002 |
| ARRAYSIZE     | 0.0000015 |
| PACK     | 0.00007 |
| UNPACK     | 0.00007 |
| PICKITEM     | 0.0027 |
| SETITEM     | 0.0027 |
| NEWARRAY     | 0.00015 |
| NEWSTRUCT     | 0.00015 |
| NEWMAP     | 0.000002 |
| APPEND     | 0.00015 |
| REVERSE     | 0.000005 |
| REMOVE     | 0.000005 |
| HASKEY     | 0.0027 |
| KEYS     | 0.000005 |
| VALUES     | 0.00007 |
| THROW     | 0.0000003 |
| THROWIFNOT     | 0.0000003 |

*Click [here](../../cn/虚拟机) to see the Chinese edition of the NeoVM*