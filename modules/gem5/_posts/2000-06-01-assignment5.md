---
Author: Jason Lowe-Power
Editor:  Maryam Babaie, Mahyar Samani
Title: ECS 201A Assignment 5
---

## Assignment 5 -- Due 11:59 pm (PST) *{{ site.data.course.dates.dino_5 }}*

### Table of Contents

- [Administriva](#administrivia)
- [Introduction](#introduction)
- [Workload](#workload)
- [Real hardware experiments](#real-hardware-experiments)
- [Experimental setup](#experimental-setup)
- [Analysis and simulation](#analysis-and-simulation)
- [Submission](#submission)
- [Grading](#grading)
- [Academic misconduct reminder](#academic-misconduct-reminder)
- [Hints](#hints)

## Administrivia

You can submit your report in pairs. Make sure to start early and post any questions you might have on Piazza. The standard late assignemt policy applies.

Use [classroom: assignment 5]() to create an assignment. You will be asked to **join**/**create** an assignment. If your teammate has already created an assignment, please **join** their assignment instead of creating one assignment. Otherwise, **create** your assignment and ask your teammate to **join** the assignment.

## Introduction

In this assignment, you will explore the performance bottlenecks in poorly-written parallel code.
We will take a very simple application, summing the values in an array, and see how if you are not careful how you parallelize the application, the performance will become quite poor.

Then, after seeing which algorithms perform well and poorly on real hardware, we will use a cycle-level simulator ([gem5](https://www.gem5.org/)) with a detailed cache model to understand the performance.

## Workload

We will explore a very simple application for this assignment: summing an array.

```c
for (int i=0; i < length; i++) {
    *result += array[i];
}
```

We will look at 6 different parallel implementations.

### Implementation 1: Naive

In the first implementation, we will assign each thread subsequent items in the array to add and have all threads *share* the result.
In other words, the first item will be summed by the first thread, the second item by the second thread, the third item by the third thread, and so on.
If we have more items than threads (of course we will! There will be more than just 4-16 items!), then a particular thread will be responsible for summing items that are `thread_id = item (mod num_thread)`

![algorithm 1 visualization]({{ '/img/parallel-alg-1.png' | relative_url }})

In this picture, each thread is represented as a different color.
The iterations shown are the different iterations of the loop below (and in `sum_1` in `parallel.cpp`).
Each square represents one integer, though in future pictures, the spaces between where the threads are accessing may not be drawn to scale.

```c++
for (int i=tid; i < length; i += threads) {
    *result += array[i];
}
```

In all of these examples `tid` is the thread id (starting at `0` up to the `threads - 1`),
`threads` is the number of threads that we're using, and `length` is the number of elements in the array.
Also, in all examples we will assume that the threads can *race* on the `result` so we must declare it as a `std::atomic` to make sure that all accesses are completed consistently.

You can find the compiled binary for this implementation in X86 under `workloads/array_sum/naive-native`.
You can use this binary to run the program on real hardware.
Here is an example of how you could run the binary on native hardware.
This example sums up an array of size `32768 elements` with `8 threads`.

```shell
./naive-native 32768 8
```

**CAUTION**: You **SHOULD NOT** run `workloads/array_sum/naive-gem5` on real hardware.

You can import this implementation to your configuration file from `workloads/array_sum_workload.py` as `NaiveArraySumWorkload`.
To instantiate an object of this workload you need to pass **array_size** and **num_threads** as arguments to `__init__`.
Here is an example of how you should create an object of this workload.
This example creates a workload of this binary that sums up `16384 elements` with `4 threads`.

```python
from workloads.array_sum_workload import NaiveArraySumWorkload

workload = NaiveArraySumWorkload(16384, 4)
```

To build the native binary for this implementation run the following command in `workloads/array_sum`.

```shell
make naive-native
```

To build the gem5 binary for this implementation run the following command in `workloads/array_sum`.

```shell
make naive-gem5
```

### Implementation 2: Chunking the array

One problem with Implementation 1 is that each array will read in a cache line worth of data (64B), but it will only use 1 value from that cache line (4B).
Thus, we are wasting 60B out of 64B or 94% of the data we're reading from memory!

To improve this (think about what kind of locality this is... see [Question 1](#question-1) below), we can instead "chunk up" the array.
We can split the array into a number of contiguous chunks equal to the number of threads and then assign each thread to one of those chunks.
This is shown visually below with the code as well.

![algorithm 2 visualization]({{ '/img/parallel-alg-2.png' | relative_url }})

```c++
size_t chunk_size = (length+threads-1)/threads;
for (int i=tid*chunk_size; i < (tid+1)*chunk_size && i < length; i++) {
    *result += array[i];
}
```

Note that we may have not quite even chunks.
This is one downside of chunking this way.
An alternative implementation would be to combine implementation 1 and chunking (e.g., chunk with some granularity of 1000 or 2000 or something).
This would reduce the *imbalance* in the last chunk.
OpenMP's `schedule static` does something like this.
However, for this assignment, it won't make much difference since the array size is much larger than the number of threads.

You can find the compiled binary for this implementation in X86 under `workloads/array_sum/chunking-native`.
You can use this binary to run the program on real hardware.
Here is an example of how you could run the binary on native hardware.
This example sums up an array of size `32768 elements` with `8 threads`.

```shell
./chunking-native 32768 8
```

**CAUTION**: You **SHOULD NOT** run `workloads/array_sum/chunking-gem5` on real hardware.

You can import this implementation to your configuration file from `workloads/array_sum_workload.py` as `ChunkingArraySumWorkload`.
To instantiate an object of this workload you need to pass **array_size** and **num_threads** as arguments to `__init__`.
Here is an example of how you should create an object of this workload.
This example creates a workload of this binary that sums up `16384 elements` with `4 threads`.

```python
from workloads.array_sum_workload import ChunkingArraySumWorkload

workload = ChunkingArraySumWorkload(16384, 4)
```

To build the native binary for this implementation run the following command in `workloads/array_sum`.

```shell
make chunking-native
```

To build the gem5 binary for this implementation run the following command in `workloads/array_sum`.

```shell
make chunking-gem5
```

### Implementation 3: Let's not be so racy

You may notice that a potential problem with implementation 1 and implementation 2 is that all of the threads are accessing the *exact same address* at the *exact same time*.
Because we declared the variable as `std::atomic` and we have smart hardware (i.e., cache coherence), we *will* end up with the correct final answer.
However, it may not perform the best.

Implementation 3 instead has a *different address* for every thread to write its result.
Now, all of the threads are fully independent!
This implementation has an array of results of length `n`, where `n` is the number of threads.
Each thread uses its thread id (`tid`) to select which result to use.
Otherwise, this implementation is the same as *implementation 1*.

Once all threads have computed their part of the result, in the `main` function, the main thread sums up all of the results.
While this is a bit of serialization (remember Amhahl!), we hope that the work from all the other threads outweighs this work significantly.

You can see this implementation visually and in code below.

![algorithm 3 visualization]({{ '/img/parallel-alg-3.png' | relative_url }})

```c++
for (int i=tid; i < length; i += threads) {
    result[tid] += array[i];
}
```

You can find the compiled binary for this implementation in X86 under `workloads/array_sum/res-race-opt-native`.
You can use this binary to run the program on real hardware.
Here is an example of how you could run the binary on native hardware.
This example sums up an array of size `32768 elements` with `8 threads`.

```shell
./res-race-opt-native 32768 8
```

**CAUTION**: You **SHOULD NOT** run `workloads/array_sum/res-race-opt-gem5` on real hardware.

You can import this implementation to your configuration file from `workloads/array_sum_workload.py` as `NoResultRaceArraySumWorkload`.
To instantiate an object of this workload you need to pass **array_size** and **num_threads** as arguments to `__init__`.
**NOTE**: Make sure to use the same number of threads as your processor cores.
This example creates a workload of this binary that sums up `16384 elements` with `4 threads`.

```python
from workloads.array_sum_workload import NoResultRaceArraySumWorkload

workload = NoResultRaceArraySumWorkload(16384, 4)
```

To build the native binary for this implementation run the following command in `workloads/array_sum`.

```shell
make res-race-opt-native
```

To build the gem5 binary for this implementation run the following command in `workloads/array_sum`.

```shell
make res-race-opt-gem5
```

### Implementation 4: Combining 2 and 3

This implementation is a straightforward combination of the optimizations in implementations 2 and 3.
We use chunking and then split up the results and use different addresses.
See the code below.

![algorithm 4 visualization]({{ '/img/parallel-alg-4.png' | relative_url }})

```c++
size_t chunk_size = (length+threads-1)/threads;
for (int i=tid*chunk_size; i < (tid+1)*chunk_size && i < length; i++) {
    result[tid] += array[i];
}
```

You can find the compiled binary for this implementation in X86 under `workloads/array_sum/chunking-res-race-opt-native`.
You can use this binary to run the program on real hardware.
Here is an example of how you could run the binary on native hardware.
This example sums up an array of size `32768 elements` with `8 threads`.

```shell
./chunking-res-race-opt-native 32768 8
```

**CAUTION**: You **SHOULD NOT** run `workloads/array_sum/chunking-res-race-opt-gem5` on real hardware.

You can import this implementation to your configuration file from `workloads/array_sum_workload.py` as `ChunkingNoResultRaceArraySumWorkload`.
To instantiate an object of this workload you need to pass **array_size** and **num_threads** as arguments to `__init__`.
Here is an example of how you should create an object of this workload.
This example creates a workload of this binary that sums up `16384 elements` with `4 threads`.

```python
from workloads.array_sum_workload import ChunkingNoResultRaceArraySumWorkload

workload = ChunkingNoResultRaceArraySumWorkload(16384, 4)
```

To build the native binary for this implementation run the following command in `workloads/array_sum`.

```shell
make chunking-res-race-opt-native
```

To build the gem5 binary for this implementation run the following command in `workloads/array_sum`.

```shell
make chunking-res-race-opt-gem5
```

### Implementation 5: Knowing the cache

We have one final optimization that we can apply to this code.
You hopefully remember that the cache access granularity is a *block* (e.g., 64B).
So, even if you only want 4B of data, you have to get an entire 64B block.

So, what we're going to do is to make sure that the data that is accessed frequently by each thread is in a *different* cache block.
Since we know that each integer is 4B, we are going to space out our accesses by 16 integers ($$ 16 \times 4 = 64 $$).
See the code based on implementation 3 below.

![algorithm 5 visualization]({{ '/img/parallel-alg-5.png' | relative_url }})

```c++
for (int i=tid; i < length; i += threads) {
    result[tid*16] += array[i];
}
```

You can find the compiled binary for this implementation in X86 under `workloads/array_sum/block-race-opt-native`.
You can use this binary to run the program on real hardware.
Here is an example of how you could run the binary on native hardware.
This example sums up an array of size `32768 elements` with `8 threads`.

```shell
./block-race-opt-native 32768 8
```

**CAUTION**: You **SHOULD NOT** run `workloads/array_sum/block-race-opt-gem5` on real hardware.

You can import this implementation to your configuration file from `workloads/array_sum_workload.py` as `NoCacheBlockRaceArraySumWorkload`.
To instantiate an object of this workload you need to pass **array_size** and **num_threads** as arguments to `__init__`.
Here is an example of how you should create an object of this workload.
This example creates a workload of this binary that sums up `16384 elements` with `4 threads`.

```python
from workloads.array_sum_workload import NoCacheBlockRaceArraySumWorkload

workload = NoCacheBlockRaceArraySumWorkload(16384, 4)
```

To build the native binary for this implementation run the following command in `workloads/array_sum`.

```shell
make block-race-opt-native
```

To build the gem5 binary for this implementation run the following command in `workloads/array_sum`.

```shell
make block-race-opt-gem5
```

### Implementation 6: With blocking ...

And finally, we can combine implementation 5 with blocking.

![algorithm 3 visualization]({{ '/img/parallel-alg-6.png' | relative_url }})

```c++
size_t chunk_size = (length+threads-1)/threads;
for (int i=tid*chunk_size; i < (tid+1)*chunk_size && i < length; i++) {
    result[tid*16] += array[i];
}
```

You can find the compiled binary for this implementation in X86 under `workloads/array_sum/all-opt-native`.
You can use this binary to run the program on real hardware.
Here is an example of how you could run the binary on native hardware.
This example sums up an array of size `32768 elements` with `8 threads`.

```shell
./all-opt-native 32768 8
```

**CAUTION**: You **SHOULD NOT** run `workloads/array_sum/all-opt-gem5` on real hardware.

You can import this implementation to your configuration file from `workloads/array_sum_workload.py` as `ChunkingNoBlockRaceArraySumWorkload`.
To instantiate an object of this workload you need to pass **array_size** and **num_threads** as arguments to `__init__`.
Here is an example of how you should create an object of this workload.
This example creates a workload of this binary that sums up `16384 elements` with `4 threads`.

```python
from workloads.array_sum_workload import ChunkingNoBlockRaceArraySumWorkload

workload = ChunkingNoBlockRaceArraySumWorkload(16384, 4)
```

To build the native binary for this implementation run the following command in `workloads/array_sum`.

```shell
make all-opt-native
```

To build the gem5 binary for this implementation run the following command in `workloads/array_sum`.

```shell
make all-opt-gem5
```

## Real hardware experiments

On a computer *with at least 4 cores* (preferably 8 or more) run the different parallel algorithms to sum an array described above.
You can run `grep -m1 "cpu cores" /proc/cpuinfo` to find out how many cores you have.
It should say something like `cpu cores       : 8`

Note: we will only support running on x86 Linux machines.
If you want to use a different system, we probably won't be able to help, and you're not guaranteed the correct results.

On your *real hardware* run the 6 parallel algorithms with 1, 2, 4, 8, 16 threads (up to the maximum threads on your hardware).
I.e., if you only have 4 cores, don't run 8 and 16.

Use the execution time measured by your binary to answer the following questions.

### Question 1

For algorithm 1, does increasing the number of threads improve performance or hurt performance? Use data to back up your answer.

### Question 2

(a) For algorithm 6, does increasing the number of threads improve performance or hurt performance? Use data to back up your answer.

(b) What is the speedup when you use 2, 4, 8, 16 threads (only answer with up to the number of cores on your system).

### Question 3

(a) Using the data for all 6 algorithms, what is the most important optimization, chunking the array, using different result addresses, or putting padding between the result addresses?

(b) Speculate how the hardware implementation is causing this result. What is it about the hardware that causes this optimization to be most important?

## gem5

For those of you who are not familiar with [gem5](https://www.gem5.org/), gem5 is a *cycle-level* simulator that simulates the entire system (cores, memory, caches, devices) at the hardware level.

The *input* to gem5 is a *Python* script which configures the system and runs the simulation.
Refer to [Assignment 0]({{'modules/gem5/assignment0' | relative_url}}) to learn how to create your own configuration script.

<!-- **For 201A** *DO NOT USE YOUR gem5 BINARY!*

I found a bug in gem5 when creating this assignment.
Some important statistics (e.g., everything in Ruby...) were not being reset correctly.
The binary included in the template has the bug fixed. -->

### gem5's output

The stats file generated by gem5 is very large and has detailed statistics for *everything* in the system.
We don't need to look at all of the statistics to try to figure out what's going on in these different algorithms.
Here, we'll concentrate on a few key stats: the total time to run the region of interest (performance), the average latency for L1 misses, and the *reasons* for the L1 misses.

For *reasons* (which I won't get into), you should ignore all of the controllers numbered `0` (e.g., ignore `board.cache_hierarchy.ruby_system.l1_controllers0`).

## Experimental setup

For this assignment, you will be the same components across your experiments.
However, for parts of the assignment you might want to change the number of processor cores or the latency of a crossbar in your cache interconnect.
Refer to the list below for more information on the components you will be using.

- boards: you will only use `HW5X86Board`.
You can find its definition in `components/boards.py`.
- processors: you will only use `HW5O3CPU`.
You can find its definition in `components/processors.py`.
**NOTE**: you will notice that the component creates a processor with an extra core.
This is a weird gem5 thing.
Please ignore this.
However, when you look at your statistics you should ignore statistics for `board.processor.core.cores0` and
`board.cache_hierarchy.ruby_system.l1_controllers0`.
- cache hierarchies: you will only use `HW5MESITwoLevelCacheHierarchy`.
You can find its definition in `components/cache_hierarchies.py`.
**NOTE**: you will notice that its `__init__` takes **two** arguments.
Make sure to pass the *number of processor cores* as `num_l2_banks`.
You will have to assign different values to `xbar_latency` as instructed in the later parts of this assignment.
- memories: You will only use `HW5DDR4`.
You can find its definition in `components/memories.py`.
- clock frequency: Use `3GHz` as your clock frequency.

## Analysis and simulation

Now, we are going to use a software simulation framework to look at the details of how the hardware operates to actually answer the "speculation" part of [question 3](#question-3).

In order to run your experiments, create a configuration script that allows you to run *any of the 6 implementations* of the workload with *any number of cores* for `HW5O3CPU` with *any latency* for `xbar_latency` in `HW5MESITwoLevelCacheHierarchy`.

### **IMPORTANT NOTE**

In your configuration scripts, make sure to import `exit_event_handler` using the command below.

```python
from workloads.roi_manager import exit_event_handler
```

You will have to pass `exit_event_handler` as a keyword argument named `on_exit_event` when creating a `simulator` object. Use the *template* below to create a simulator object.

```python
simulator = Simulator(board={name of your board}, full_system=False, on_exit_event=exit_event_handler)
```

### Performance

To get the performance/time of the region of interest, you can see the 3rd line in the stats file: `simSeconds`.
This is the **simulated time** or the time that the *simulator* says the program *will* take.
(As opposed to the `hostSeconds` which is how long it took to simulate on your host.)
This simulated time should be the same order of magnitude as what you say on your hardware, around 1 millisecond.

### Cache behavior

Let me warn you, these stats are confusing!
However, hopefully we can narrow them down to make sense of them.

#### Hits and misses

First, let's look at cache hits and misses.
In our system, we may have a bunch of difference caches.
They will be named something like `board.cache_hierarchy.ruby_system.l1_controllers2.L1Dcache` and `board.cache_hierarchy.ruby_system.l1_controllers15.L1Dcache` if, for instance, you use 16 cores (there's one L1 cache per core).

**IMPORTANT:** Ignore `board.cache_hierarchy.ruby_system.l1_controllers0`!! This is *not* a "real" core. (For *reasons*... let's not talk about it. This is a weird gem5 thing.)

For each cache, there is a statistic named `m_demand_hits` which counts the number of hits and a statistic named `m_demand_misses` which counts the number of misses.

So, if you search the `stats.txt` file for `board.cache_hierarchy.ruby_system.l1_controllers2.L1Dcache.m_demand_hits` you should see the number of misses for the second L1 Cache controller's L1 data cache.

#### L1 miss latency

As we've talked about in class, the latency for L1 misses is an important statistic.
Unsurprisingly, like everything else in gem5, we have a stat for that!
If you search the file for `m_missLatencyHistSeqr` you will find a confusingly represented histogram for the miss latencies.
To keep things simple, you can just worry about the *average* miss latency, which can be found with `board.cache_hierarchy.ruby_system.m_missLatencyHistSeqr::mean`.
(On a side note, this is aggregated across all L1 caches, but that's fine for the way we're going to use the average.)

#### Average memory access time

Not only does gem5 track the times for hits and misses separately, it also has a statistic for the latency for *all* accesses.
For the average memory latency, you can use `board.cache_hierarchy.ruby_system.m_latencyHistSeqr::mean`.
This statistic looks the same as the `m_missLatencyHistSeqr`, but it includes both hits and misses.

#### Read sharing

We briefly talked in class about different *coherence* protocols.
In the system we're simulating in this gem5 simulation, the caches follow a "MESI" coherence protocol.
So, a cache block can be "read shared" between many different L1 caches.

(Unsurprisingly) there's a statistic for the number of times a block *becomes* shared: `board.cache_hierarchy.ruby_system.L1Cache_Controller.Fwd_GETS`.

Now, we need to talk about the confusing way this statistic is presented.
Here's an example from using 16 cores with algorithm 1.

```text
board.cache_hierarchy.ruby_system.L1Cache_Controller.Fwd_GETS |         349     14.55%     14.55% |        1940     80.87%     95.41% |           7      0.29%     95.71% |           8      0.33%     96.04% |           8      0.33%     96.37% |           8      0.33%     96.71% |           8      0.33%     97.04% |           8      0.33%     97.37% |           7      0.29%     97.67% |           8      0.33%     98.00% |           9      0.38%     98.37% |           8      0.33%     98.71% |           8      0.33%     99.04% |           8      0.33%     99.37% |           6      0.25%     99.62% |           4      0.17%     99.79% |           5      0.21%    100.00% (Unspecified)
```

The pipe symbols (`|`) differentiate the events from *different* controllers.
In this example, there are 16 L1 caches (well, 17, but we are ignoring `0`), so you can see many different entries.
Each entry shows both a number and the percentage of the total (sum) and the cumulative percentage.
You can ignore the percentages.

So, you should **ignore the first entry** because this is a fake/annoying/unrealistic L1.
After that first ignored entry, you'll see the L1 controller number 1 has 1,940 `Fwd_GETS` events.
This means that 1,940 times another cache wanted to get the data this cache has in shared state.
This is a measure of the *read sharing* of this algorithm.

#### Write "sharing"

Like read sharing we may be interested in what happens with data that is shared between caches, but it is *written*.
To see the number of times that some cache had to respond to another cache asking to have the data in *writable* state, you can see the `Fwd_GETX` statistic.
I.e., you can use `board.cache_hierarchy.ruby_system.L1Cache_Controller.Fwd_GETX`.
This stat has the exact same format as the `Fwd_GETS` described above.

### Getting ready to answer questions

To dig into why we're seeing the performance results on the real hardware, with gem5 we can look at more extreme systems.
Instead of just using 4 or 8 cores, let's use 16!

For 16 cores, run *each algorithm* and save the stats output.
I strongly recommend using `--outdir` and using easy to understand names.
The following questions will ask you to consult these outputs and use data to back up your answers.

### Question 4

(a) What is the speedup of algorithm 1 and speedup of algorithm 6 on *16 cores* as estimated by gem5?

(b) How does this compare to what you saw on the real system?

### Question 5

Which optimization (chunking the array, using different result addresses, or putting padding between the result addresses) has the biggest impact on the *hit ratio?*

Show the data you use to make this determination.
Discuss which algorithms you are comparing and why.

### Question 6

Which optimization (chunking the array, using different result addresses, or putting padding between the result addresses) has the biggest impact on the *read sharing?*

Show the data you use to make this determination.
Discuss which algorithms you are comparing and why.

### Question 7

Which optimization (chunking the array, using different result addresses, or putting padding between the result addresses) has the biggest impact on the *write sharing?*

Show the data you use to make this determination.
Discuss which algorithms you are comparing and why.

### Question 8

Let's get back to the question we're trying to answer. From [question 3](#question-3) above, "What is it about the hardware that causes this optimization to be most important?"

So:
(a) Out of the three characteristics we have looked at, the L1 hit ratio, the read sharing, or the write sharing, which is most important for determining performance?
Use the average memory latency (and overall performance) to address this question.

Finally, you should have an idea on what optimizations have the biggest impact on the hit ratio, the read sharing performance, and the write sharing performance.

So:
(b) Using data from the gem5 simulations, now answer what hardware characteristic *causes* the most important optimization to be the most important?

### Question 9

**NOTE**: This question is for 201A students **only**.

Run using a crossbar latency of 1 cycle and 25 cycles (in addition to the 10 cycles that you have already run).

As you increase the cache-to-cache latency, how does it affect the importance of the different optimizations?

You don't have to run all algorithms.
You can probably get away with just running algorithm 1 and algorithm 6.

## Submission

As mentioned before, you are allowed to submit your assignments in **pairs** and in **PDF** format.
You should submit your report on [gradescope](https://www.gradescope.com/courses/487868).
In your report answer the questions presented in , [Question 1](#question-1), [Question 2](#question-2),[Question 3](#question-3),[Question 4](#question-4),[Question 5](#question-5),[Question 6](#question-6),[Question 7](#question-7),[Question 8](#question-8), and [Question 9](#question-9).

Use clear reasoning and visualization to drive your conclusions.

Submit all your code through your assignment repository. Please make sure to include code/scripts for the following.

- `Instruction.md`: should include instruction on how to run your simulations.
- Automation: code/scripts to run your simulations.
- Configuration: python file configuring the systems you need to simulate.

## Grading

Like your submission, your grade is split into two parts.

1. Reproducibility Package (50 points):
    a. Instruction and automation to run simulations for different section and dump statistics (10 points)
        - Instructions (5 points)
        - Automation (5 points)
    b. configuration script(s) (40 points)
2. Report (50 points): the grading breakdown is as follows (they are subject to change, but they show the relative breakdown) :

Total points = 50

| #Question       | Points |
|-----------------|--------|
| Question 1	    | 2.5      |
| Question 2.a	  | 2.5	     |
| Question 2.b	  | 2.5      |
| Question 3.a    | 5     |
| Question 3.b    | 2.5      |
| Question 4.a	  | 2.5      |
| Question 4.b	  | 2.5      |
| Question 5	    | 5     |
| Question 6	    | 5     |
| Question 7	    | 5     |
| Question 8.a	  | 5     |
| Question 8.b	  | 5     |
| Question 9	    | 5     |

## Academic misconduct reminder

You are required to work on this assignment in teams. You are only allowed to share you scripts and code with your teammate(s). You may discuss high level concepts with others in the class but all the work must be completed by your team and your team only.

Remember, DO NOT POST YOUR CODE PUBLICLY ON GITHUB! Any code found on GitHub that is not the base template you are given will be reported to SJA. If you want to sidestep this problem entirely, don’t create a public fork and instead create a private repository to store your work.

## Hints

- Start early and ask questions on Piazza and in discussion.
- If you need help, come to office hours for the TA, or post your questions on Piazza.
