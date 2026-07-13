---
title: A summary of the modern CPU
draft: false
---
# Instructions from the processor's POV

Imagine the following instruction:

```asm
movzx eax, BYTE PTR [rbx]
```

Where `rbx` contains some memory address.

From our perspective, this is a single instruction. The CPU fetches it and then executes it, easy as that.

But in reality, there are a lot more details than that.

If you think about the above instruction, what does it do? It doesn't all magically happen in one step.

Let's assume it does the following in order:
It reads the value from memory then zero-extends it and finally writes the result back to `eax`.

So a single instruction gets broken down into one or more smaller operations that are executed by the processor.

Those smaller operations are called **micro-operations**, usually denoted as `uops`.

The act of breaking an instruction down into micro-operations is known as **decoding**.

Conceptually, we can say that the above instruction gets decoded into:
- `uop1`: read value from memory
- `uop2`: zero-extend that value
- `uop3`: write the result back to register `eax`

Notice that in our flow of operations here, `uop3` can't be executed until `uop2` finishes, which in turn can't be executed until `uop1` finishes. In other words `uop3` depends on the result of `uop2` which depends on the result of `uop1`.

We can try and visualize it better like so:
```
uop1
  ↓
uop2
  ↓
uop3
```

They form something called a **dependency chain**.
# The Pipeline

So far, we saw the instruction going through three "stages". It was fetched then decoded and finally executed:

![[/imgs/Pasted image 20260712181146.png|340]]

All these different "stages" are a part of something called the **CPU pipeline**.

Let's expand our code a little and add another instruction:
```asm
inst1: movzx eax, BYTE PTR [rbx]
inst2: add rdi, QWORD PTR [0x1337]
```

As we now know, `inst1` goes through the fetching stage. When that's done, it can then go through the decoding stage.

But wait, the fetching stage is empty now while our goal should be to maximize the CPU's performance/output.\
The processor can take advantage of that empty stage by beginning to fetch `inst2` while `inst1` is being decoded.

Now both `inst1` and `inst2` are inside the CPU which is against our original belief that the CPU works on one instruction at a time.

![[/imgs/Pasted image 20260712181111.png|340]]

Without going into a lot of details, it's worth noting that the execution stage in the pipeline itself is made up of different execution units, each responsible for different kinds of work such as arithmetic operations, memory access operations, etc.\
Many of these execution units are themselves pipelined.
This means that more than one `uop` can be inside the execution stage at the same time, whether at the same execution unit (since it's pipelined) or a different one.

# Dependency Chain
Let's use the same example from before but change the register in the second instruction:
```asm
inst1: movzx eax, BYTE PTR [rbx]
inst2: add rax, QWORD PTR [0x1337]
```

From an abstracted point of view, we can say that `inst2` needs the value in the register `rax` that comes from `inst1`. This is *somewhat* true. Because if we zoom in a little and look at the `uops` that make up these instructions, we get a little bit of a different picture.

Like before, we can conceptually say that `inst1` get decoded into:
- `uopX1`: read value from memory
- `uopX2`: zero-extend that value
- `uopX3`: write the result back to register `eax`

And `inst2` becomes:
- `uopY1`: read value from memory
- `uopY2`: read value from register `rax`
- `uopY3`: add both values into register `rax`

We can say that *most* of `inst2` depends on `inst1`, except for `uopY1` which can be executed without needing any values from the other `uops`.

Now our chain looks something like this:
```
uopX1
  ↓
uopX2
  ↓
uopX3
  ↓
uopY2      uopY1
  |          |
  |          |
  |          |
   ----|----
       V
     uopY3
```

So it's important to keep in mind that dependency chains happen at the `uop` level, between different `uops` of different instructions.
# Bubbles
Now that we know that multiple `uops` can be inside the execution stage at the same time, let's go take another look at our first example:
- `uop1`: read value from memory
- `uop2`: zero-extend that value
- `uop3`: write the result back to register `eax`
```
uop1
  ↓
uop2
  ↓
uop3
```
There's a dependency here, as we now know, between the `uops` so they can't being execution at the same time; `uop3` needs values from `uop2` that needs values from `uop1`.

We have to keep in mind that not all operations are the same, nor do they take the same amount of time to finish executing or produce a value.

`uop1` retrieves a value from memory and memory-related operations relatively take more time than something like an addition operation, since it needs to go read the value from somewhere else.

Why am I mentioning this?\
Because we just said that `uop2` depends on `uop1`, it can't begin execution at the same time as it.
So `uop2` has to wait until `uop1` produces a result, creating a gap between them where nothing is happening.

This gap is known as a **bubble**.

But we have mentioned before that we should maximize the performance/output of the CPU, a bubble goes against our desire in this case, we can't just leave the CPU idle.\
This is where **out-of-order execution** comes into play.

# Out-of-order Execution

## Out-of-order
For the next couple of sections, let's make things easier a bit by assuming that each instruction as a whole either depends on a value from a previous instruction or it doesn't.

Let's go back to our example and add one more instruction:
```asm
inst1: movzx eax, BYTE PTR [rbx]
inst2: add rax, QWORD PTR [0x1337]
inst3: mov rax, 10
```

Following our assumption,  `inst2` depends on `inst1` and we need to fill in the bubble-ed space between them so as not to leave any stage idle.

We can do that by bringing in *another* instruction, that has no dependency on either of them, and place it between them.

This is what **out-of-order execution** is all about.

Can we use `inst3` to do that?\
Keep in mind that we can not put an instruction that has a dependency on either `inst1` or `inst2`. The CPU might look at it and see `rax` and say no, `inst3` also needs `rax`.
But this is not true (*false*) since we are not *using* the value in `rax`, we are assigning a new one. This is known as a **false dependency**.

But wait, wouldn't executing `inst3` before `inst2` make `inst2` use an incorrect value?

No! It all works out in the end due to **register renaming**.
## Register Renaming
Get ready, because if you have been reading assembly for a while now, this is going to blow your mind.

You know those CPU registers that we often see while we go through the disassembly of a binary:
`rdi, rsi, rdx, rax` and so on.

They're not "real" in the physical sense. Sit with that for a minute.

They are merely an abstraction, mapped to actual *physical* registers in the CPU.
So for example:
```
rdi -> Physical Register 14
rsi -> Physical Register 45
rdx -> Physical Register 90
```
Where the registers `rdi, rsi, rdx` get *renamed* to those physical registers at some point when instructions enter the pipeline. That renaming changes constantly.

Why is this important?

All of these instructions might be sharing the same "architectural" register name but they are mapped to different physical registers inside the CPU and those physical registers are the ones that hold the actual values that we need.

So for our last example:
```asm
inst1: movzx eax, BYTE PTR [rbx]
inst2: add rax, QWORD PTR [0x1337]
inst3: mov rax, 10
```
It's absolutely fine to run `inst3` before `inst2` since each one of those `rax` registers are mapped to different physical registers.

Pretty cool huh?
# Speculative Execution
## Branch Prediction
Now what if `inst3` in our last example was a branching instruction?

Let's apply that and also change a couple of things:
```asm
inst1: movzx eax, BYTE PTR [rbx]
inst2: add rax, QWORD PTR [0x1337]
inst3: cmp rax, 2
inst4: je DOUBLE
inst5: add rax, 4
       DOUBLE:
inst6: add rax, 8
```

Here we can see that we jump to the branch called `DOUBLE` based on some condition, in our case if the register `rax` contains the value 2. If it doesn't we continue on with our execution of `inst5`.

Recall how we mentioned before that `inst1` is a memory operation instruction and those usually take a long time (relatively) so we would like to move on and see what else we can execute while it finishes.

But the catch here is the CPU doesn't yet know what to execute; which branch to choose. It doesn't yet have the value of `rax` to be able to determine the outcome of the condition in `inst3`.
So what it does is it *speculates* the outcome of that comparison and chooses a branch to execute. This is <u>a type of</u> **speculative execution** called **branch prediction** and sometimes **control-flow speculation**.

There is a dedicated [branch predictor](https://en.wikipedia.org/wiki/Branch_predictor) inside the CPU that is responsible for these conditions but I won't go into details about it here.

So the CPU makes a "prediction" and chooses a branch and executes it. Let's assume here the CPU guesses that `inst5` was going to run.

A good question arises here is what happens if this prediction was wrong? The CPU guessed the outcome and it found out later that it chose the wrong path.

The CPU simply discards the results of its guess and rewinds back its state to what it was before it made the guess. `inst5` is now called a **transient instruction** since it was mispredicted that it was going to be executed and only existed for the time that the CPU guessed it was going to execute.

However, a small problem here occurs when not all results could be ignored.
For example, if a transient instruction accesses memory, some data may be brought into 
the cache and its data would live on.

---
Now an important thing to note here is that speculative execution is not limited to branch prediction, branch prediction is merely a type of speculative execution.

Sources:
- https://www.lighterra.com/papers/modernmicroprocessors/
- https://www.agner.org/optimize/microarchitecture.pdf
- https://en.wikipedia.org/wiki/Micro-operation
- https://www.geeksforgeeks.org/computer-organization-architecture/computer-organization-micro-operation/
- https://pages.cs.wisc.edu/~fischer/cs701.f14/lectures/L18.pdf
- https://www.intel.com/content/www/us/en/developer/articles/technical/software-security-guidance/technical-documentation/hardware-behavior-related-to-speculative-execution.html
- https://www.intel.com/content/www/us/en/developer/articles/technical/software-security-guidance/best-practices/refined-speculative-execution-terminology.html

