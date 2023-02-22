---
Author: Hoa Nguyen, Jason Lowe-Power
Editor: Hoa Nguyen
Title: DINO CPU Assignment 4
---

DINO CPU Assignment 4: The Why of Caching.

Originally from ECS 154B Lab 4, Winter 2023.

## Table of Contents

* [Introduction](#introduction)
* [Updating DINO CPU on GitHub Codespaces / CSIF machines](#updating-dino-cpu-on-github-codespaces--csif-machines)
* [Glossary](#glossary)
* [The Computer System](#the-computer-system)
  * [The Core](#the-core)
  * [The Cache System](#the-cache-system)
    * [DINOCPU Cache Details](#dinocpu-cache-details)
  * [The RAM Device](#the-ram-device)
* [The Benchmarks](#the-benchmarks)
* [Part I: Implementing the Hazard Detection Unit for Non Combinational Pipelined CPU](#part-i-implementing-the-hazard-detection-unit-for-non-combinational-pipelined-cpu)
  * [The new memory interface](#the-new-memory-interface)
  * [The contract between the core and the Memory Interface](#the-contract-between-the-core-and-the-memory-interface)
  * [Updating the Pipelined CPU](#updating-the-pipelined-cpu)
  * [Debugging the Hazard Unit](#debugging-the-hazard-unit)
  * [Testing the Non Combinational Pipelined CPU](#testing-the-non-combinational-pipelined-cpu)
* [Part II: Performance Evaluation](#part-ii-performance-evaluation)
  * [Question 1 (10 points)](#question-1-10-points)
  * [Question 2 (20 points)](#question-2-20-points)
  * [Question 3 (15 points)](#question-3-15-points)
  * [Question 4 (15 points)](#question-4-15-points)
* [Part III: Performance Analysis](#part-iii-performance-analysis)
  * [Question 5 (10 points)](#question-5-10-points)
  * [Question 6 (10 points)](#question-6-10-points)
* [Logistics](#logistics)
  * [Grading](#grading)
  * [Submission](#submission)
* [Extra Credits (10 points)](#extra-credits-10-points)
* [Conclusion](#conclusion)

## Introduction

Caching is among the most influential ideas in computing.
In fact, the concept reaches many fields of computer science.
Examples can be found in from almost all hardware architectures, such as the
idea of translation lookup buffers (TLBs) or the use of memory cache system, to
various software designs, such as DNS caching or web caching.

In essence, the idea of caching is to store previously acquired data in a cache
that is close to the component uses the data.
The idea utilizes the assumption that the cost of making a copy of the data is
low, as well as the the cost of querying for the data in a cache is less
expensive compared to acquring the data again.
As a result, if the data is frequently requested, the cache could help bringing
down the average cost of requesting for that data.
In other words, we are trading the cost of acquiring the data with 
*fast storage* capacity.

However, caching does not inherently improve the performance of a system.
Caching adds an extra cost of querying for the existence of data in the cache,
and since the capacity of a cache is limited, the cache system needs extra
work to maintain the entries in the cache.
Due to physical constraints, such as area, power, and latency, the cache closer
to the core tends to have drastically smaller capacity than the ones further
away from the core.
Thus, designing a performant memory cache in a core imposes a huge challenge
to the designers.

In this assignment, we will investigate the performance of a computer system
with a pipelined CPU core, a memory system, and various cache design decisions.
The assignment is designed as follows: we will introduce each of the components
of the computer system that we are going to investigate, then we will introduce
the benchmarks that will be subsequently used for performance evaluation and
performance analysis.
In terms of implementation, most of the system is implemented.
However, we will ask you to complete the hazard detection unit for the new
pipelined CPU.

## Updating DINO CPU on GitHub Codespaces / CSIF machines
The GitHub Classroom page for the class is located at [{{site.data.course.154b_github_classroom_link}}]({{site.data.course.154b_github_classroom_link}}).

The assignment 4 template repo is located at [{{site.data.course.154b_assignment4_github_link}}]({{site.data.course.154b_assignment4_github_link}}).

Follow the following link to access assignment 4: [{{site.data.course.154b_assignment4_invitation_link}}]({{site.data.course.154b_assignment4_invitation_link}}).

The above link will automatically create a repo in the GitHub Classroom page that only you have the access to.

In the event that the template repo is updated, your own repo won't be automatically updated.
You don't need to keep track of the template repo, unless we found an error in the assignment, in which case, we will make an announcement on {{site.data.course.discussion_site}} and provide ways to update your repo.

## Glossary

* Core/CPU terminology: In technical terms, the cache system resides on a CPU.
We will use the term "core" to refer to the pipeline, and the term "CPU" to
refer to the pipeline and the cache system.
* Memory Latency: The time between when a memory device receives a request
and when it completes the request.
* Memory Hierarchy: A series of memory devices that, the lower level a memory
device is, the higher memory latency it has.
* Memory System: All memory devices including the cache system and RAM.
* Memory Transaction: A memory device can send a *memory request* to its lower
next memory device, which will send back the corresponding *memory response*
when the request is complete. The DINOCPU memory system returns 8 bytes of
requested data for each memory read request.
* Dirty Cache Entry: A cache entry is written to at some point that has yet to
be written back to the lower level memory.
* Cache Entry Eviction: When there is a cache miss, a cache system will have to
send a memory request to get the data to the cache. Upon receiving the
response, if all cache entry are occupied, one of the cache *current* entry
will be evicted, i.e., this entry will be sent to the next lower
level memory if the next lower level memory does not have the most recent
value of the dirty entry. The evicted entry will be replaced by the new data.
* LRU replacement policy: When all cache entry are occupied, the cache entry
with that was least recently accessed.

## The Computer System

![system\_diagrams]({{'img/dinocpu/assignment4-systems.svg' | relative_url}})
**Figure 1.** Illustration of the systems that we use for evaluating different
memory cacheh structures.
On each system, the left arrows are wires responsible for sending instruction memory
requests/responses, while the right arrows are wires responsible for sending data
memory requests/responses.

### The Core

We will use the same pipeline that we built in the assignment 3.
However, the memory interface is slightly tweaked to support memory devices
of which the latency of a memory request is unknown to the core
[\[1\]](#cache-miss-latency-might-vary).

Note that both instructions and data come from memory devices.
As a result, extra care should be taken to ensure correct instructions enter
the pipeline, as well as the correct data are received.
For this assignment, you will update the hazard unit to make sure that the
the core receives correct instructions/data from the cache system.

### The Cache System

We will use one-level cache system for this assignment.
We call this a split L1 Cache.

The split L1 Cache consists of two components: an L1 Instruction Cache
(L1I) and L1 Data Cache (L1D).
The Instruction Cache is optimized for read-only accesses.
The Data Cache supports both read and write operations.
Each of the L1I and L1D is 4-way associative cache having 32 entries.
The cache block size is 8 bytes.
Each cache uses the LRU replacement policy, and the write-back write policy.

### DINOCPU Cache Details

Each cache component consists of one `CacheTable` and one `CacheMemory`.

A `CacheMemory` is an array of registers containing all cache blocks.
Note that `CacheMemory` only has the data, it doesn't have metadata.

A `CacheTable` is a table of `CacheEntry`s, where each `CacheEntry` contains
the metadata uniquely identifies a cache block, as well as metadata indicating
the status of the cache block, and a pointer to the cache block data in the
`CacheMemory`.

A CacheEntry is structured as followed,

```scala
class CacheEntry(val numEntryIndexingBits:Int, val numTagBits: Int) extends Bundle {
  val valid  = Bool()                       // Whether this entry contains valid data.
  val dirty  = Bool()                       // Whether this entry is written to by the core. Default to False, set to True upon a write request.
  val tag    = UInt(numTagBits.W)           // Contains the tag bits.
  val memIdx = UInt(numEntryIndexingBits.W) // Where is it in the cache memory.
  val age    = UInt(32.W)                   // When was the most recent access to this cache block.
}
```

For example,

```scala
idx 27 -> CacheEntry(valid ->  1, dirty ->  1, tag -> 211, memIdx -> 27, age -> 17614)
```

means,

* `valid ->  1`: The entry contains valid data.
* `dirty ->  1`: The entry was written to.
* `tag -> 211`: The tag bits are 211 in decimal, or 0xd3 in hexadecimal.
* `memIdx -> 27`: The 28th entry of CacheMemory has data of this cache block.
* `age -> 17614`: The most recent access to this cache block was in the cycle of 17614.

### The RAM Device

The RAM Device is the lowest level memory device in the DINOCPU system.
All accesses to the RAM device incurs a 30 CPU cycles latency.

## The Benchmarks

For this assignment, we evaluate the effectiveness of the cache system using
a benchmark called `stream`, which is inspired by
[the popular STREAM benchmark](https://www.cs.virginia.edu/stream/).

The workloads are named as followed,

```
stream-<n>-stride-<stride>[-noverify].riscv
```

where,
* each element is of size 16 bits.
* `n`: number of memory accesses *per* array *per* iteration.
* `stride`: the distance between two consecutive elements.
* `-noverify`: if the workload verifies the result array.

For this assignment, we will use `stream-64-stride-1.riscv` and
`stream-64-stride-4.riscv` for verifying the correctness of the
pipelined implentation. The `stream-64-stride-1-noverify.riscv`
and `stream-64-stride-4-noverify.riscv` are used for performance
evaluation.

![benchmark]({{'img/dinocpu/assignment4-stream-for-loop.svg' | relative_url}})
**Figure 2.** The main part of the stream benchmarks consists of calling the `copy()`
function twice. When running each of the benchmarks, the CPU spends most time
executing the instruction inside the `copy()` function.

## Part I: Implementing the Hazard Detection Unit for Non Combinational Pipelined CPU

In this part, you will complete the hazard detection unit for the non combinational
pipelined CPU in `components/hazardnoncombin.scala`.
The descriptions on top of the file that file should give explanation for the meaning
of each signal.
The following subsections describe the changes in the cores as well as in the memory.
Note that you don't have to change any component other than the hazard unit for the
non combinational core.

### The new memory interface

![mem\_interface\_diagrams]({{'img/dinocpu/assignment4-mem-interface.svg' | relative_url}})

**Figure 3.** The new memory interface, which includes a bundle of valid/ready/good signals
that are used for synchronization between the core and the memory subsystem. Both `imem` and
`dmem` use the new memory interface. Other than the new signals, the input signals and output
signals of the memory are the same. E.g., for the instruction memory interface, the memory
request signal consists of the `address` signal, and the memory response signal consists of
`instruction` signal.

### The contract between the core and the Memory Interface

* The core only sends a request when Memory is ready, and the core sets the valid signal to true.
* The memory only processes one request at a time.
* If the memory receives a request in the current cycle, it will start processing the request in the
next cycle.
* If the memory is not ready, then even if the CPU sets valid to 1, the memory does not receive
the request.
* There is no mechanism for cancelling a memory request that was received by a memory component.
* The memory only guarantees the response to be correct if and only if the `good`
signal is 1. This means, if the `good` signal is 0, the response might contain garbage.
* The `good` signal is only set to one for 1 cycle. This means, the response is only guaranteed to
be correct in that cycle.
* A load instruction will result in a read memory request being sent to the memory subsystem.
* A store instruction will result in a read memory request and a write memory request being
sent to the memory subsystem.

### Updating the Pipelined CPU

The pipelined CPU that is compatible with non combinational memory components is located
at `pipelined/cpu-noncombin.scala` file.
You don't have to modify the `cpu-noncombin.scala` file.

The DINO CPU have a new memory interface for both `imem` and `dmem` for this
assignment.

The code for the Non Combination Pipelined CPU is mostly the same as the Pipelined CPU.
However, we are going to use the `HazardUnitNonCombin` rather than `HazardUnit`.

### Debugging the Hazard Unit

There are tools implemented for debugging the hazard unit,

- You can use the single stepper for debugging using the following commands,

```sh
runMain dinocpu.singlestep <test_name> 1 pipelined-non-combin
```

- You can uncomment this line
[here]({{site.data.course.154b_assignment4_github_link}}/blob/main/src/main/scala/pipelined/cpu-noncombin.scala#L128)
of `cpu-noncombin.scala` to see the PCs of the instructions in the pipeline for each cycle.

- You can uncomment this code block
[here]({{site.data.course.154b_assignment4_github_link}}/blob/main/src/main/scala/pipelined/cpu-noncombin.scala#L131)
of `cpu-noncombin.scala` to see the sequence of PCs of the instructions that are in the writeback stage.

For debugging the hazard unit when running large applications, it is useful to compare
the committed instruction traces of a benchmark when running the single-cycle cpu and
the pipelined-non-combin cpu.

An instruction is "committed" means that instruction is complete its execution.
For the single-cycle cpu, since it executes all instructions in 1 cycle, every
instruction the cpu executes are committed.
In a 5-stage pipelined cpu, it means the instruction goes through all 5 stages.
Since if an instruction ever reaches the WB stage, it will complete all of its
5 stages.
Therefore, you can track all instructions reached the WB stage and see the sequences
of instructions being committed.
Since each binary has one program order regardless of the microarchitectures, if the
5-stage cpu is correctly implemented, its commit trace must be exactly the same
as the single-cycle cpu commit trace for every binary.

By comparing the traces, you can figure out the problematic cycle, and use the single stepper
to step to that cycle and see what happened in that cycle.

### Hints

* The instruction outputted by imem is only correct when imem's good signal is 1.
* Similarly, the readdata outputted by dmem is only correct when dmem's good signal is 1.
* There's no new control hazard or new data hazard. We are modifying the hazard unit
because we can stall and flush stage registers using the hazard unit. In other words,
the control hazards and data hazards are the same as in the previous assignments.
* One way to think about this assignment is to list all combinations of hazards and timing
signals and decide what to do in each scenario.
* It's easier to start with making sure only correct instructions enter the pipeline.
Meaning, only advance an instruction from IF stage to ID stage when imem's good signal
is set.
* There are multiple ways of handling control hazards and data dependencies. You can follow
discussion sessions for suggested ways. However, the suggestions might not be the most
optimal. You are encouraged to discuss the timing in the pipeline with your
peers/TAs/instructor. You must write the code yourself, however.
* There's no guarantee in terms of memory request latency when there is a cache miss.
Therefore, the CPU should solely rely on ready/good signals to decide when to send a
memory request and to receive a memory response. Explanation for those signals can be
found in the discussions slides.

### Testing the Non Combinational Pipelined CPU

```scala
Lab4 / testOnly dinocpu.SmallTestsTesterLab4
Lab4 / testOnly dinocpu.FullApplicationsTesterLab4
```

To run a test,

```scala
runMain dinocpu.simulate_predefined_system <system_name> <riscv_binary>
```

where,
`<system_name>` is one of `default`, `system1`, `system2`, `system3`, and `system4`.
The `default` system should be used for testing purposes.
The `default` system is structurally the same as `system1` except that the RAM latency
is 1 cycle instead of 30 cycles in other systems.

**Notes:**

* The tests on Gradescope are **without** caches.
* For `SmallTestsTesterLab4` tests: It is expected that, if you are using a system
**with** data caches, some of the store tests, ones start with `s` prefix, will fail
because there are data in the cache that are not written back to the RAM yet!
Those store tests only check the data in RAM rather than in cache, hence the test
failures.
* For `FullApplicationsTesterLab4` tests: This test suite includes
`stream-64-stride-1.riscv` and `stream-64-stride-4.riscv`, which are used to ensure
the benchmarks we used in Part II and Part III run correctly.

## Part II: Performance Evaluation

For this assignment, we will consider 4 systems,

* System 1: Non Combinational Pipelined CPU + No Cache
* System 2: Non Combinational Pipelined CPU + Instruction Cache (No Data Cache)
* System 3: Non Combinational Pipelined CPU + Data Cache (No Instruction Cache)
* System 4: Non Combinational Pipelined CPU + Instruction Cache + Data Cache

To run a benchmark on a system,

```scala
runMain dinocpu.simulate_predefined_system <system_name> <riscv_binary>
```

where,

* `<system_name>` is one of `default`, `system1`, `system2`, `system3`, and `system4`.
The `default` system should be used for testing purposes.
The `default` system is structurally the same as `system1` except that the RAM latency
is 1 cycle instead of 30 cycles in other systems.
* `<riscv_binary>` is name of the benchmark.

**Notes:** Since there are multiple ways of dealing with timing in a CPU, each
correct implementation might slightly differs in terms of the optimality, and
thus we don't expect to see exactly the same amount of cycles per each data point
when comparing your implementation with the solution.
However, as long as the correctness is ensured, the general trend of the number of
cycles should not differ. I.e., there are some systems that are strictly slower than
others.
Therefore, we opt to use graphs rather than exact numbers for reporting data.

**Notes:** You don't have to report raw data or how the calculation was done
for question 2, question 3, and question 4.
If you are unsure about the correctness of the graph, you can show the formula
that you used to generate the data for the graph.
However, the graphs must have the X-axis and Y-axis with labels and units.

### Question 1 (10 points)

Determine the number of dynamic instructions of the
`stream-64-stride-1-noverify.riscv` and the
`stream-64-stride-4-noverify.riscv`.

**Hint:** Regardless of microarchitectures, the amount of dynamic instruction
must be the same for each binary compiled using the RV64IM instruction set.
The single cycle CPU might be useful for this question.

**Hint:** DINOCPU simulations output the number of cycles, and does not output
the number of simulated instructions. Part of the Iron Law might be useful for
the calculation step.

### Question 2 (20 points)

Create a graph representing the CPI of the Non Combinational CPU in 4 systems
described above with the `stream-64-stride-1-noverify.riscv` benchmark and the
`stream-64-stride-4-noverify.riscv` benchmark.
The X-axis should be grouped by systems.

![CPI\_graph]({{'img/dinocpu/assignment4-CPI-graph-example.svg' | relative_url}})
**Figure 4.** An example of a CPI graph with the collected data. The
data on the graph are not the real CPI of running the mentioned benchmarks on
the four systems.

### Question 3 (15 points)

Assume that the pipelined non combinational CPU is clocked at 2.5GHz.
Create a graph illustrating the effective bandwidth of system 4 when running
each of `stream-64-stride-1-noverify.riscv` and
`stream-64-stride-4-noverify.riscv` benchmarks.

Effective bandwidth is defined as the amount of **data** used by the
core per second.
For example, when the core executes an `lw` instruction, it uses 4 bytes
(32 bits) of data in that cycle.

**Note:** The unit can be bytes/second, or a multiple of bytes/second, such as
KiB/second and MiB/second.

### Question 4 (15 points)

Create a graph representing the L1 data cache hit ratio and the L1 instruction
cache hit ratio when running each of `stream-64-stride-1-noverify.riscv` and
`stream-64-stride-4-noverify.riscv` benchmarks with system 4.

## Part III: Performance Analysis

### Question 5 (10 points)

Between data cache and instruction cache, do you think which cache has more
impact on performance? Explain why using the data from part II.

### Question 6 (10 points)

From the data from Part II, you should see system 4 performs better
when running `stream-64-stride-1-noverify.riscv` compared to running
`stream-64-stride-4-noverify.riscv`. Explain why using the data from part II.

## Logistics

### Grading
Part I will be automatically graded on Gradescope.
See the Submission section for more details.

| Part                | Percentage |
|---------------------|------------|
| Part 4.1            |        20% |
| Part 4.2 Question 1 |        10% |
| Part 4.2 Question 2 |        20% |
| Part 4.2 Question 3 |        15% |
| Part 4.2 Question 4 |        15% |
| Part 4.3 Question 5 |        10% |
| Part 4.3 Question 6 |        10% |
| Extra Credits       |        10% |

### Submission

**Warning:** read the submission instructions carefully. Failure to adhere to the instructions will result in a loss of points.

#### Code Portion

You will upload the following files to Gradescope on the [Assignment 4 - Code Portion]({{site.data.course.154b_gradescope_lab4_code_link}}) assignment,

* `src/main/scala/pipelined/cpu-noncombin.scala`

Once uploaded, Gradescope will automatically download and run your code.
This should take less than 40 minutes.
For each part of the assignment, you will receive a grade.
If all of your tests are passing locally, they should also pass on Gradescope unless you made changes to the I/O, which you are not allowed to do.

**Note:** Make sure that you comment out or remove all printing statements in your submissions.
There are a lot of long tests for this assignment.
The auto-grader might fail complete grading within the allocated time if there are too many output statements.

#### Written Portion

You will upload your answers for the [Assignment 4 - Written Portion]({{site.data.course.154b_gradescope_lab4_written_link}}) assignment to Gradescope.
Please upload a separate page for each answer!
Additionally, I believe Gradescope allows you to circle the area with your final answer.
Make sure to do this!

We will not grade any questions for which we are unable to read.
Be sure to check your submission to make sure it's legible, right-side-up, etc.

### Academic misconduct reminder

You are to work on this project **individually**.
You may discuss *high level concepts* with one another (e.g., talking about the diagram), but all work must be completed on your own.

**Remember, DO NOT POST YOUR CODE PUBLICLY ON GITHUB!**
Any code found on GitHub that is not the base template you are given will be reported to SJA.
If you want to sidestep this problem entirely, don't create a public fork and instead create a private repository to store your work.
GitHub now allows everybody to create unlimited private repositories for up to three collaborators, and you shouldn't have any collaborators for your code for this assignment.

### Hints

* **Start early!** Completing part 4.1 is essential for the next parts.
* If you need help, come to office hours for the TAs, or post your questions on {{site.data.course.discussion_site}}.


## Extra Credits (10 points)

Improve the performance of the non combination pipelined CPU when running *any* of
application in the Full Application tests and their `-loops-unrolled` variants.
You can find the tests [here]({{site.data.course.154b_assignment4_github_link}}/blob/main/src/main/scala/testing/InstTests.scala#L1096)
and [here]({{site.data.course.154b_assignment4_github_link}}/blob/main/src/main/scala/testing/InstTests.scala#L1198).

The new CPU performance should match one of the following cases,

1. General performance improvement across most full application tests.
It's okay to have a marginal performance improvement in this case.
2. The CPU performs especially well (>1.1x speedup) on a couple of full applications,
while suffers slowdowns on other full applications. Those designs are called
application accelerators.

Constraints:

* You are not allowed to modify the latency or capacity of the existing memory
components.
* You are also not allowed to change the memory hierarchy, such as adding another
level of cache.

However, you can make any changes to the core such as adding more components to the core.
You also can change the internal of the memory components, such as changing the
cache replacement policy, or allowing the memory interface to send a response and
receive a request within the same cycle, or optimize the state diagram of the cache
component.

To submit the extra credits portion, email either TAs or the instruction with the
following files,

* A zip file containing all files that you modified.
* A pdf report describing what you did to improve the performance of the CPU, and
the speedups/slowdowns achieved from the Full Application tests.

## Conclusion

The assignment should show that, on a realistic setup, it is very hard to
keep all stages of the 5-stage pipeline busy when every memory access is
expensive.
Even with fast caches, keeping the IPC for *this* pipeline close to the ideal 1.0
is unattainable if you want to keep the CPU frequency high.

Not only does the 5-stage pipeline suffer from the problem of memory being
slower than the core, but to keep the pipeline as busy as possible,
modern architectures have a fetch stage with ability to fetch and issue multiple
instructions at a time, and speculatively fetches instructions to instruction cache,
and fetches data to data cache.
These techniques, along with multi-issue, multi-way execution, and using load/store
queues, are for exploiting ILP.

However, exploiting ILP can only help improving the performance so much
until the CPU has to run a memory-intensive applications.
Even if a CPU can hold limited amount of load/store instructions at a time,
there's a limit on how many requests can be processed in parallel.
In the next assignment, you will explore a case where, even though the
applications are memory-intensive, if the data can be independently processed,
there are DLP techniques in hardware that helps increasing the throughput of memory
accesses.

On the hardware/software interaction side, for this assignment, the simple
benchmarks should reveal the effectiveness of the cache system on different
program behaviors.
Having a cache system, even a simple one like in the DINO CPU, drastically
improves the performance of the system on a lot of real world applications.
That is why even a low-end chip has some short of a cache system, and why
a high-end chip tends to dedicate a lot of its area for the cache system.

However, a cache system does not always improve the performance for all
workloads.
In fact, by introducing a cache system, we are making the latency of pulling
data from memory higher.
There are algorithms that are naturally cache unfriendly, and it is
still an open question on whether the cache system should be able to adapt to
a variety algorithms, or the software developers should reprogram the program
to maximize the utilization of the cache system.

On the other hand, as you can see from the assignment, the timing within a computer
system is significantly dependent on the behavior of workload itself, and a small
change in a system, like a slightly larger cache size, might significantly change
the performance of a system.
Thus, performance evaluation of a new system is usually heavily relied on
simulations.
For the next assignment, you'll investigate the performance of a system with an
even more sophisticated cache system where the cache system has to handle
accesses from multiple CPU cores while having to ensure the correctness of the
program.
We will use another simulator, gem5, which comes with a variety of cache
coherency protocols allowing investigating the performance of a multiple cores
system.

## Footnotes

### Cache miss latency might vary

[1] This is not because the latency of a memory device is hidden from the CPU.
This is a complication caused by internal activities of a memory device that
are abstracted away by the CPU/Memory interface.
Consider a cache miss in an L1D cache which causes the L1D cache to pull a
cache block from memory to L1D cache.
However, since the cache is full, it needs to evict an entry to memory.
Note that, the cache sends to response to the CPU as soon as it sends the
eviction request to memory.
In the next instruction, there could be an L1D cache miss again.
However, the memory might be busy with the previous eviction request, so the
L1D cache must wait for the memory to complete that request before it can
pull a cache block to memory.
Thus, the cache miss latency might vary from access to access.
