---
Author: Jason Lowe-Power
Editor:  Maryam Babaie
Title: ECS 201A Assignment 5
---

## Assignment 5 -- Due 11:59 pm (PST) *{{ site.data.course.dates.dino_5 }}*

### Table of Contents

- [Introduction](#introduction)
- [Application](#application)
  - [Implementation 1: Naive](#implementation-1-naive)
  - [Implementation 2: Chunking the array](#implementation-2-chunking-the-array)
  - [Implementation 3: Let's not be so racy](#implementation-3-lets-not-be-so-racy)
  - [Implementation 4: Combining 2 and 3](#implementation-4-combining-2-and-3)
  - [Implementation 5: Knowing the cache](#implementation-5-knowing-the-cache)
  - [Implementation 6: With blocking...](#implementation-6-with-blocking)
  - [Template files](#template-files)
- [Real hardware experiments](#real-hardware-experiments)
  - [Question 1](#question-1)
  - [Question 2](#question-2)
  - [Question 3](#question-3)
- [gem5 experiments](#gem5-experiments)
  - [A bit about gem5](#a-bit-about-gem5)
  - [Using gem5](#using-gem5)
  - [gem5's output](#gem5s-output)
    - [Performance](#performance)
    - [Cache behavior](#cache-behavior)
      - [Hits and misses](#hits-and-misses)
      - [L1 miss latency](#l1-miss-latency)
      - [Average memory access time](#average-memory-access-time)
      - [Read sharing](#read-sharing)
      - [Write "sharing"](#write-sharing)
  - [Getting ready to answer questions](#getting-ready-to-answer-questions)
  - [Question 4](#question-4)
  - [Question 5](#question-5)
  - [Question 6](#question-6)
  - [Question 7](#question-7)
  - [Question 8](#question-8)
  - [Question 9 (**201A only**)](#question-9-201a-only)
- [Submission](#submission)
- [Grading](#grading)
- [Academic misconduct reminder](#academic-misconduct-reminder)
- [Additional notes](#additional-notes)

## Introduction

In this assignment, you will explore the performance bottlenecks in poorly-written parallel code.
We will take a very simple application, summing the values in an array, and see how if you are not careful how you parallelize the application, the performance will become quite poor.

Then, after seeing which algorithms perform well and poorly on real hardware, we will use a cycle-level simulator ([gem5](https://www.gem5.org/)) with a detailed cache model to understand the performance.

## Application

We will explore a very simple application for this assignment: summing an array.

```c
for (int i=0; i < length; i++) {
    *result += array[i];
}
```

We will look at 6 different parallel implementations (called `sum_<1-6>` in the template C++ code).

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

### Implementation 6: With blocking...

And finally, we can combine implementation 5 with blocking.

![algorithm 3 visualization]({{ '/img/parallel-alg-6.png' | relative_url }})

```c++
size_t chunk_size = (length+threads-1)/threads;
for (int i=tid*chunk_size; i < (tid+1)*chunk_size && i < length; i++) {
    result[tid*16] += array[i];
}
```

### Template files

You can download the template files [here]({{ '/img/assignment5-template.tgz' | relative_url }}).


  - The 6 different algorithms. You can compile this to generate `parallel` binaries: `assignment5-template/parallel.cpp`

  - The gem5 run script: `assignment5-template/run.py`

  - The `parallel` binary to run on real hardware: `assignment5-template/parallel-x86`

  - The `parallel` binary to use with gem5. Same as `parallel-x86` except for the gem5-specific special instructions around the region of interest: `assignment5-template/parallel-x86-gem5`

  - The files for the coherence protocol configuration: `assignment5-template/my_mesi/`

  - The gem5 binary: `assignment5-template/gem5-x86`

## Real hardware experiments

On a computer *with at least 4 cores* (preferably 8 or more) run the different parallel algorithms to sum an array described above.
You can run `grep -m1 "cpu cores" /proc/cpuinfo` to find out how many cores you have.
It should say something like `cpu cores       : 8`

Note: we will only support running on x86 Linux machines.
If you want to use a different system, we probably won't be able to help, and you're not guaranteed the correct results.

A binary is already built for you and distributed in the [template files](#template-files).
If you want to build it yourself, you can use the following command.

```sh
g++ -o parallel-x86 parallel.cpp -lpthread
```

This program takes 3 command line parameters: The size of the array, the number of threads to use, and the algorithm to use (as defined above).
There is a limitation that you cannot use an array larger than 65536 as the integers will overflow.

On your *real hardware* run the 6 parallel algorithms with 1, 2, 4, 8, 16 threads (up to the maximum threads on your hardware).
I.e., if you only have 4 cores, don't run 8 and 16.

Use the performance shown by the `parallel-x86` (`Time 0.XXXX ms`) app to answer the following questions.

### Question 1

For algorithm 1, does increasing the number of threads improve performance or hurt performance? Use data to back up your answer.

### Question 2

(a) For algorithm 6, does increasing the number of threads improve performance or hurt performance? Use data to back up your answer.

(b) What is the speedup when you use 2, 4, 8, 16 threads (only answer with up to the number of cores on your system).

### Question 3

(a) Using the data for all 6 algorithms, what is the most important optimization, chunking the array, using different result addresses, or putting padding between the result addresses?

(b) Speculate how the hardware implementation is causing this result. What is it about the hardware that causes this optimization to be most important?

## gem5 experiments

Now, we are going to use a software simulation framework to look at the details of how the hardware operates to actually answer the "speculation" part of [question 3](#question-3).

In the [template files](#template-files) is everything you need to run gem5, including the gem5 binary (`gem5-x86`).

### A bit about gem5

For those of you who are not familiar with [gem5](https://www.gem5.org/), gem5 is a *cycle-level* simulator that simulates the entire system (cores, memory, caches, devices) at the hardware level.

The *input* to gem5 is a *Python* script which configures the system and runs the simulation.
I have also given you this Python script (`run.py`) and other extra Python files to configure the system (`riscv_se.py` and the files in `my_mesi/`).

This Python script configures gem5 to look like the drawing below, and it runs the `parallel-x86-gem5` binary.
As part of the configuration the script sets up the command line parameters for the benchmark application, so you don't have to worry about that.
This new binary (`parallel-x86-gem5`) is just like the one you ran on real hardware (`parallel-x86`), but it has a couple of special instructions to tell the simulator when the *region of interest* begins and ends.

> **SIDE NOTE**
> If you want to build this binary yourself, you can use the following command (assuming that the root gem5 directory is `../gem5` and you have built the `m5` library.)
> `g++ -o parallel-x86-gem5 parallel.cpp -lpthread -D GEM5 -I../gem5/include -L../gem5/util/m5/build/x86/out/ -lm5`

Like the timing code (e.g., timer start and timer stop) that's used to measure the *time* of the region of interest on real hardware, the annotations added for gem5 reset the statistics and dump the statistics at the beginning and end of the region of interest, respectively.

**For 201A** *DO NOT USE YOUR gem5 BINARY!*

I found a bug in gem5 when creating this assignment.
Some important statistics (e.g., everything in Ruby...) were not being reset correctly.
The binary included in the template has the bug fixed.

### Using gem5

To use the `run.py` script, there are three parameters: the number of cores in the system to simulate (this will allow you to simulate up to 16 cores), the algorithm that the `parallel` program should use (this is passed to the benchmark application), and a "crossbar latency" (`xbar_latency`).
For now, ignore the last parameter.

To run gem5, you can do something like below:

```sh
./gem5-x86 run.py 1 1 10
```

You can also run `gem5-x86 run.py --help` for details on the command line parameters.

*Note: Don't forget to give a proper full path for run.py file in your command.*

When you run gem5, an output directory is created (`m5out/` by default) which holds a file called `stats.txt` which has way too many detailed statistics.
I would suggest using an extra command line parameter `--outdir=<name>` to save your output to different directories (otherwise, gem5 will just overwrite the last simulation).
For example, the code below will create a `stats.txt` file in `<cwd>/m5out/1-core-1-alg/`.
Note that I used the parameters I passed to `run.py` to help name the output directory.

```sh
./gem5-x86 --outdir=m5out/1-core-1-alg run.py 1 1
```

### gem5's output

The stats file generated by gem5 is very large and has detailed statistics for *everything* in the system.
We don't need to look at all of the statistics to try to figure out what's going on in these different algorithms.
Here, we'll concentrate on a few key stats: the total time to run the region of interest (performance), the average latency for L1 misses, and the *reasons* for the L1 misses.

For *reasons* (which I won't get into), you should ignore all of the controllers numbered `0` (e.g., ignore `board.cache_hierarchy.ruby_system.l1_controllers0`).

#### Performance

To get the performance/time of the region of interest, you can see the 3rd line in the stats file: `simSeconds`.
This is the **simulated time** or the time that the *simulator* says the program *will* take.
(As opposed to the `hostSeconds` which is how long it took to simulate on your host.)
This simulated time should be the same order of magnitude as what you say on your hardware, around 1 millisecond.

#### Cache behavior

Let me warn you, these stats are confusing!
However, hopefully we can narrow them down to make sense of them.

##### Hits and misses

First, let's look at cache hits and misses.
In our system, we may have a bunch of difference caches.
They will be named something like `board.cache_hierarchy.ruby_system.l1_controllers2.L1Dcache` and `board.cache_hierarchy.ruby_system.l1_controllers15.L1Dcache` if, for instance, you use 16 cores (there's one L1 cache per core).

**IMPORTANT:** Ignore `board.cache_hierarchy.ruby_system.l1_controllers0`!! This is *not* a "real" core. (For *reasons*... let's not talk about it. This is a weird gem5 thing.)

For each cache, there is a statistic named `m_demand_hits` which counts the number of hits and a statistic named `m_demand_misses` which counts the number of misses.

So, if you search the `stats.txt` file for `board.cache_hierarchy.ruby_system.l1_controllers2.L1Dcache.m_demand_hits` you should see the number of misses for the second L1 Cache controller's L1 data cache.

##### L1 miss latency

As we've talked about in class, the latency for L1 misses is an important statistic.
Unsurprisingly, like everything else in gem5, we have a stat for that!
If you search the file for `m_missLatencyHistSeqr` you will find a confusingly represented histogram for the miss latencies.
To keep things simple, you can just worry about the *average* miss latency, which can be found with `board.cache_hierarchy.ruby_system.m_missLatencyHistSeqr::mean`.
(On a side note, this is aggregated across all L1 caches, but that's fine for the way we're going to use the average.)

##### Average memory access time

Not only does gem5 track the times for hits and misses separately, it also has a statistic for the latency for *all* accesses.
For the average memory latency, you can use `board.cache_hierarchy.ruby_system.m_latencyHistSeqr::mean`.
This statistic looks the same as the `m_missLatencyHistSeqr`, but it includes both hits and misses.

##### Read sharing

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

##### Write "sharing"

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

### Question 9 (**201A only**)

Run using a crossbar latency of 1 cycle and 25 cycles (in addition to the 10 cycles that you have already run).

As you increase the cache-to-cache latency, how does it affect the importance of the different optimizations?

You don't have to run all algorithms.
You can probably get away with just running algorithm 1 and algorithm 6.

## Submission

For this assignment you don't need to turn in any code files.
The only file you need to submit on gradescope at the designated section, is the pdf file of your report.
Please do not forget to specify each question according to the outline when you're submitting your work.
In your report, you're not required to include any data which is not used in your analysis.
Only include those that you actually use to justify your answer and make sure they are precisely and clearly specified.

**Note: if the submission does not have *page match* for each question, a *9-points penalty* will be applied to the assignment5's grade**

## Grading

The grading breakdown is as follows (they are subject to change, but they show the relative breakdown) :

Total points = 100

| #Question       | Points |
|-----------------|--------|
| Question 1	    | 5      |
| Question 2.a	  | 5	     |
| Question 2.b	  | 5      |
| Question 3.a    | 10     |
| Question 3.b    | 5      |
| Question 4.a	  | 5      |
| Question 4.b	  | 5      |
| Question 5	    | 10     |
| Question 6	    | 10     |
| Question 7	    | 10     |
| Question 8.a	  | 10     |
| Question 8.b	  | 10     |
| Question 9	    | 10     |

## Academic misconduct reminder

You are to work on this project **individually**.
You may discuss *high level concepts* with one another (e.g., talking about the diagram), but all work must be completed on your own.

**Remember, DO NOT POST YOUR CODE PUBLICLY ON GITHUB!**
Any code found on GitHub that is not the base template you are given will be reported to SJA.
If you want to sidestep this problem entirely, don't create a public fork and instead create a private repository to store your work.
GitHub now allows everybody to create unlimited private repositories for up to three collaborators, and **you shouldn't have *any* collaborators** for your code in this class.

## Additional notes
* Start early! There is a learning curve for gem5, so start early and ask questions on Campuswire and in discussion.

* If you need help, come to office hours for the TA, or post your questions on Campuswire.

