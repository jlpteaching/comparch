---
Author: Jason Lowe-Power
Editor:  Maryam Babaie, Mahyar Samani, Kaustav Goswami
Title: Exploring false sharing with Ruby cache coherence models
---

**Due on *{{ site.data.course.dates.gem5_6 }}* 11:59 pm (PST)**: See [Submission](#submission) for details

GitHub Classroom link for 154B: [{{ site.data.course.154b_assignment6_invitation_link }}]({{ site.data.course.154b_assignment6_invitation_link }})

GitHub Classroom link for 201A: [{{ site.data.course.201a_assignment6_invitation_link }}]({{ site.data.course.201a_assignment6_invitation_link }})

### Table of Contents

- [Introduction](#introduction)
- [Workload](#workload)
- [Real hardware experiments](#real-hardware-experiments)
- [Experimental setup](#experimental-setup)
- [Analysis and simulation](#analysis-and-simulation)
- [Submission](#submission)
- [Grading](#grading)
- [Academic misconduct reminder](#academic-misconduct-reminder)

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

![algorithm 1 visualization](images/parallel-alg-1.png)

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

You can find the the makefile to compile the binary under `workloads/array_sum/naive-native`.
Run `make all-native` in the `workloads/array_sum` to compile the binaries.
You can use these binaries to run the program on real hardware.
Here is an example of how you could run the binary on native hardware.
This example sums up an array of size `65536 elements` with `4 threads`.

```shell
./naive-native 65536 4
```

**CAUTION**: You **SHOULD NOT** run `workloads/array_sum/naive-gem5` on real hardware.

Here is an example of how you can use this workload in gem5.
By default, this workload of this binary that sums up `32768 elements` with `16 threads`.

```python
    workload = obtain_resource(f"array_sum_naive_run")
    board.set_workload(workload)
```

This should take around 1-2 minutes.

To build the gem5 binary for this implementation run the following command in `workloads/array_sum`.

```shell
make naive-gem5
```

### Implementation 2: Chunking the array

One problem with Implementation 1 is that each array will read in a cache line worth of data (64B), but it will only use 1 value from that cache line (4B).
Thus, we are wasting 60B out of 64B, or 94% of the data we're reading from memory!

To improve this (think about what kind of locality this is... see [Question 1](#question-1) below), we can instead "chunk up" the array.
We can split the array into several contiguous chunks equal to the number of threads and then assign each thread to one of those chunks.
This is shown visually below with the code as well.

![algorithm 2 visualization](images/parallel-alg-2.png)

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
The gem5 workload is named "array_sum_chunking_run".

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

You can see this implementation visually and in the code below.

![algorithm 3 visualization](images/parallel-alg-3.png)

```c++
for (int i=tid; i < length; i += threads) {
    result[tid] += array[i];
}
```

You can find the compiled binary for this implementation in X86 under `workloads/array_sum/res-race-opt-native`.
The gem5 workload for this algorithm is named "array_sum_race_optimized_run".

### Implementation 4: Combining 2 and 3

This implementation is a straightforward combination of the optimizations in implementations 2 and 3.
We use chunking and then split up the results and use different addresses.
See the code below.

![algorithm 4 visualization](images/parallel-alg-4.png)

```c++
size_t chunk_size = (length+threads-1)/threads;
for (int i=tid*chunk_size; i < (tid+1)*chunk_size && i < length; i++) {
    result[tid] += array[i];
}
```

You can find the compiled binary for this implementation in X86 under `workloads/array_sum/chunking-res-race-opt-native`.
The gem5 workload for this algorithm is named "array_sum_chunking_race_optimized_run".

### Implementation 5: Knowing the cache

We have one final optimization that we can apply to this code.
You hopefully remember that the cache access granularity is a *block* (e.g., 64B).
So, even if you only want 4B of data, you have to get an entire 64B block.

So, what we're going to do is make sure that the data that is accessed frequently by each thread is in a *different* cache block.
Since we know that each integer is 4B, we are going to space out our accesses by 16 integers ($$ 16 \times 4 = 64 $$).
See the code based on implementation 3 below.

![algorithm 5 visualization](images/parallel-alg-5.png)

```c++
for (int i=tid; i < length; i += threads) {
    result[tid*16] += array[i];
}
```

You can find the compiled binary for this implementation in X86 under `workloads/array_sum/block-race-opt-native`.
The names for these workloads are getting a little out of control, this one is named "array_sum_result_cache_optimized_run" to use with gem5.

### Implementation 6: With blocking ...

And finally, we can combine implementation 5 with blocking.

![algorithm 3 visualization](images/parallel-alg-6.png)

```c++
size_t chunk_size = (length+threads-1)/threads;
for (int i=tid*chunk_size; i < (tid+1)*chunk_size && i < length; i++) {
    result[tid*16] += array[i];
}
```

You can find the compiled binary for this implementation in X86 under `workloads/array_sum/all-opt-native`.
This gem5-compatible workload is named "array_sum_all_optimizations_run".

## Real hardware experiments

On your *codespace* run the six different parallel algorithms with 1, 2 and 4 threads.
(If you run this on your own machine you may get different results which do not make sense.
Please use codespaces.)

Use the execution time measured by your binary to answer the following questions.

### Question 1

For algorithm 1, does increasing the number of threads improve performance or hurt performance? Use data to back up your answer.

### Question 2

(a) For algorithm 6, does increasing the number of threads improve performance or hurt performance? Use data to back up your answer.

(b) What is the speedup when you use 2, and 4 threads.

### Question 3

(a) Using the data for all 6 algorithms, what is the most important optimization, chunking the array, using different result addresses, or putting padding between the result addresses?

(b) **Create a hypothesis** for why the hardware implementation is causing this result. What is it about the hardware that causes this optimization to be most important?

### gem5's output

The stats file generated by gem5 is very large and has detailed statistics for *everything* in the system.
We don't need to look at all of the statistics to try to figure out what's going on in these different algorithms.
Here, we'll concentrate on a few key stats: the total time to run the region of interest (performance), the average latency for L1 misses, and the *reasons* for the L1 misses.

For *reasons* (which I won't get into), you should ignore all of the controllers numbered `0` (e.g., ignore `board.cache_hierarchy.ruby_system.l1_controllers0`).

## Experimental setup

For this assignment, you will use the same components across your experiments.
However, for parts of the assignment, you might want to change the number of processor cores or the latency of a crossbar in your cache interconnect.
Refer to the list below for more information on the components you will be using.

- boards: you will only use `X86Board`.
You can find its definition in `components/__init__.py`.
- processors: you will only use `O3CPU`. This processor has one parameter which is the number of cores.
You can find its definition in `components/__init__.py`.
**NOTE**: you will notice that the component creates a processor with an extra core.
This is a weird gem5 thing.
Please ignore this.
However, when you look at your statistics you should ignore statistics for `board.processor.core.cores0` and
`board.cache_hierarchy.ruby_system.l1_controllers0`.
- cache hierarchies: you will only use `MESITwoLevelCacheHierarchy`.
You can find its definition in `components/cache_hierarchies.py`.
**NOTE**: you will notice that its `__init__` takes **one** argument.
You will have to assign different values to `xbar_latency` as instructed in the later parts of this assignment.
- memories: You will only use `DDR4`.
You can find its definition in `components/__init__.py`.
- clock frequency: Use `3GHz` as your clock frequency.

## Analysis and simulation

Now, we are going to use a software simulation framework to look at the details of how the hardware operates to answer the "speculation" part of [question 3](#question-3).

To run your experiments, create a configuration script that allows you to run *any of the 6 implementations* of the workload with *16 cores* for `O3CPU`.
Later, you can add another parameter which will run the system with *any latency* for `xbar_latency` in `MESITwoLevelCacheHierarchy`.

### Performance

To get the performance/time of the region of interest, you can see the 3rd line in the stats file: `simSeconds`.
This is the **simulated time** or the time that the *simulator* says the program *will* take.
(As opposed to the `hostSeconds` which is how long it took to simulate on your host.)
This *simulated time* should be the same order of magnitude as what you say on your hardware, around 1 millisecond.

### Cache behavior

Let me warn you, these stats are confusing!
However, hopefully, we can narrow them down to make sense of them.

#### Hits and misses

First, let's look at cache hits and misses.
In our system, we may have a bunch of different caches.
They will be named something like `board.cache_hierarchy.ruby_system.l1_controllers2.L1Dcache` and `board.cache_hierarchy.ruby_system.l1_controllers15.L1Dcache` (there's one L1 cache per core).

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

Not only does gem5 track the times for hits and misses separately, but it also has a statistic for the latency for *all* accesses.
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
This means that 1,940 times another cache wanted to get the data this cache has in the shared state.
This is a measure of the *read sharing* of this algorithm.

#### Write "sharing"

Like read sharing we may be interested in what happens with data that is shared between caches, but it is *written*.
To see the number of times that some cache had to respond to another cache asking to have the data in *writable* state, you can see the `Fwd_GETX` statistic.
I.e., you can use `board.cache_hierarchy.ruby_system.L1Cache_Controller.Fwd_GETX`.
This stat has the same format as the `Fwd_GETS` described above.

### Getting ready to answer questions

To dig into why we're seeing the performance results on the real hardware, with gem5 we can look at more extreme systems.
Instead of just using 4 or 8 cores, we will use 16!
Moreover, we will use `32768` for the size of the array.
All of the workloads provided in `resources.json` that you get with `obtain_resource` have these paraemters set to 32768 and 16.

For 16 cores, run *each algorithm* and save the stats output.
I strongly recommend using `--outdir` and using easy-to-understand names.
Alternatively, you can set the output directory programatically in your run script with `simulator.override_outdir`

The following questions will ask you to consult these outputs and use data to back up your answers.

### Question 4

(a) What is the speedup of algorithm 1 and speedup of algorithm 6 on *16 cores* compared to *1 core* as estimated by gem5?

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

Finally, you should have an idea of what optimizations have the biggest impact on the hit ratio, the read sharing performance, and the write sharing performance.

So:
(b) Using data from the gem5 simulations, now answer what hardware characteristic *causes* the most important optimization to be the most important.

### Question 9

**NOTE**: This question is for 201A students **only**.

Run using a crossbar latency of 1 cycle and 25 cycles (in addition to the 10 cycles that you have already run).

As you increase the cache-to-cache latency, how does it affect the importance of the different optimizations?

You don't have to run all algorithms.
You can probably get away with just running algorithm 1 and algorithm 6.

## Submission

You will submit this assignment via GitHub Classroom.

1. Accept the assignment by clicking on the link provided in the announcement
2. Create a Codespace for the assignment on your repository
3. Fill out the `questions.md` file
4. Commit your changes

Make sure you include both your runscript, an explanation of how to use your
script, and the answers to the questions in the `questions.md` file.

### Explanation of How to Use Your Script

Include a detailed explanation of how to use your script and how you use your
script to generate your answers. Make sure that all paths are relative to this
directory (`virtual-memory/`).

- Include a description of what the script does
- Include the path to the script
- Include any parameters that need to be passed to the script
- Include example commands used to gather data
  - These should be able to be copy-pasted and run without modification
  - Include any required files in your repository

## Grading

- **25 points** gem5 runscript and explanation of how to use your script
- **50 points** for the questions in the report

Points Breakdown:

| #Question       | Points (201A) | Points (154B) |
|-----------------|--------|---|
| Question 1	    | 2.5      | 4 |
| Question 2.a	  | 2.5	     | 4 |
| Question 2.b	  | 2.5      | 4 |
| Question 3.a    | 5     | 4 |
| Question 3.b    | 2.5      | 4 |
| Question 4.a	  | 2.5      | 2.5 |
| Question 4.b	  | 2.5      | 2.5 |
| Question 5	    | 5     | 5 |
| Question 6	    | 5     | 5 |
| Question 7	    | 5     | 5 |
| Question 8.a	  | 5     | 5 |
| Question 8.b	  | 5     | 5 |
| Question 9	    | 5     | 0 |

## Academic misconduct reminder

You are required to work on this assignment in teams. You are only allowed to share your scripts and code with your teammate(s). You may discuss high-level concepts with others in the class but all the work must be completed by your team and your team only.

Remember, DO NOT POST YOUR CODE PUBLICLY ON GITHUB! Any code found on GitHub that is not the base template you are given will be reported to SJA. If you want to sidestep this problem entirely, don’t create a public fork instead create a private repository to store your work.
