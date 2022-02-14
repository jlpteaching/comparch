---
Author: Jason Lowe-Power
Editor:  Maryam Babaie
Title: ECS 201A Assignment 4
---

## Assignment 4 -- Due 11:59 pm (PST) Friday, February 25th 2022

## Table of Contents
  - [Introduction](#introduction)
  - [Application: matrix multiplication](#application-matrix-multiplication)
  - [Template files](#template-files)
  - [Compile gem5 to execute RISC-V binaries](#compile-gem5-to-execute-risc-v-binaries)
  - [Different cache designs](#different-cache-designs)
  - [Questions](#questions)
  - [Submission](#submission)
  - [Grading](#grading)
  - [Academic misconduct reminder](#academic-misconduct-reminder)
  - [Additional notes](#additional-notes)

## Introduction

You should do this assignment on your own, although you are welcome to talk to classmates in person or on Campuswire about any issues you may have encountered.
The standard late assignment policy applies.

In this assignment, you'll be investigating the performance impacts of different cache architectures and different algorithm designs on matrix multiplication.
The goals of this assignment are:

- Show how algorithms have different behaviors as the microarchitecture changes.
- Show how changing the algorithm can change performance on the *same* microarchitecture.
- Improve your understanding of cache architectures.

For this assignment, since we're interested only in the kernel of the benchmarks, we'll be using gem5's *syscall emulation* mode.
For this assignment, we don't care about I/O or operating system effects, so we will not use full system.
This means that we won't have to wait for Linux to boot or require a precompiled Linux kernel, etc.

## Application: matrix multiplication

### Base matrix multiplication algorithm

Below is a simple implementation of matrix multiplication given in `mm.cpp` file [here]({{ '/img/assignment4-template.tgz' | relative_url }}).

```cpp
void serial_multiply(double **A, double **B, double **C, int size)
{
    for (int i = 0; i < size; i++) {
        for (int j = 0; j < size; j++) {
            for (int k = 0; k < size; k++) {
                C[i][j] += A[i][k] * B[k][j];
            }
        }
    }
}
```

This algorithm is shown in the figure below.

![matrix multiplication]({{ '/img/mm.png' | relative_url }})

### Blocked matrix multiplication

You can improve the cache behavior of matrix multiplication by using a blocked algorithm.
In this algorithm, rather than streaming through all of the inputs, you operate on a *block* at a time.

Similar to loop interchange, there are multiple different ways you can choose to block the matrix multiplication algorithm.
One example is shown below where `k` and `j` are blocked and `i` is streamed.

![blocked matrix multiplication]({{ '/img/bmm.png' | relative_url }})

## Template files

In the template files, you'll find a matrix multiplication implementation which has both non-blocked and blocked algorithms.

You can run the `mm-x86` binary locally.
It takes two parameters: the size of the matrix and the blocking size.

```sh
./mm-x86 <size> <blocking>

Supported blocksizes:
0 => not blocked (default)
1 => 4x4
2 => 8x8
3 => 16x16
4 => 64x64
```

[Download the template files here.]({{ '/img/assignment4-template.tgz' | relative_url }})

### Files

- `mm.cpp`: A simple matrix multiplication code.
- `my_cache.py`: A configurable two level cache hierarchy.
- `riscv_se.py`: A simple SE mode board for RISC-V.
- `run.py`: The runscript for gem5. It takes two arguments: `<blocking type>` and `<cache_config>`. See the `--help` for more information.
    *Note*: In this file, you may need to adjust the path to `mm-riscv` at line 7 and `mm-riscv-gem5` at line 13 according to where you put your template files and gem5 root directory.
- `mm-x86`: The matrix multiply binary compiled for x86.
- `mm-riscv`: The matrix multiply binary compiled for RISC-V.
- `mm-riscv-gem5`: The matrix multiply binary with annotations for gem5 compiled for RISC-V.

## Compile gem5 to execute RISC-V binaries

First, we need to create a gem5 simulator binary that can execute RISC-V code.
As of the writing of this assignment (Winter 2022), each *target* ISA that you want to run requires a different gem5 build.
To compile gem5 to execute RISC-V binaries, we will use the default build options file for RISC-V: `build_opts/RISCV`.

**Important Note**: For this assignment you need Python 3.8 or newer. If you're using *CSIF machines or any machine that has older version of Python* (e.g., 3.6), you need to follow an additional step that we describe here, **before** compiling gem5 RISC-V binary. As shown [in this patch](https://gem5-review.googlesource.com/c/public/gem5/+/55863/2/src/python/gem5/components/boards/abstract_board.py), you are required to slightly modify the file located at `gem5/src/python/gem5/components/boards/abstract_board.py`.

1. Remove only `, final` part in line #40.
2. Remove the `@final` in line #239.

You can use the following command to build gem5's RISC-V binary.

```sh
scons build/RISCV/gem5.opt -j<your number of cores>
```

**Reminder**: If you're using CSIF machines, you should use this format: **$HOME/.local/bin/scons** for `scons`.

Note that this requires 3-4 GiB of disk space, so if you're running on a computer or virtual machine with limited disk space you may need to delete other prior builds (e.g., `rm -r build/X86`).

## Different cache designs

We have provided you with three different cache designs for both L1 and L2 caches:

- `SmallL1`: 16 KiB with 1 cycle latency
- `MediumL1`: 32 KiB with 2 cycle latency
- `LargeL1`: 64 KiB with 5 cycle latency
- `SmallL2`: 128 KiB with 10 cycle latency
- `MediumL2`: 256 KiB with 15 cycle latency
- `LargeL2`: 512 KiB with 20 cycle latency

We've also given an example of three configurations of the cache hierarchy:

```python
def get_cache_hierarchy(config):
    if config == '1':
        return My2LevelCacheHierarchy(SmallL1(), LargeL2())
    if config == '2':
        return My2LevelCacheHierarchy(MediumL1(), MediumL2())
    if config == '3':
        return My2LevelCacheHierarchy(LargeL1(), SmallL2())
```

You can modify/extend/etc. this in any way.

## Questions

### Question 1: Working set sizes

A) What is the working set size for the matrix multiply application?

B) For each of the four blocking configurations, what's the active working set for multiplying *one block*?

### Question 2: Performance

A) For the three configurations given, which cache configuration performs the best? Why? (Use AMAT to explain.)

B) For the three configurations given, which blocking scheme performs the best? Why? (Use statistics from the simulator to explain.)

### Question 3: Configuration

A) Can you find a configuration and a blocking scheme that performs best?
Don't exhaustively search, but try to use the information and the statistics from the three configurations given to find a better one.

B) What is more important, hit ratio, latency, L1, or L2 cache?

### Question 4: Running on x86

Now, run the matrix multiply on some real x86 hardware. I suggest using an input of 256 or 512.

A) What blocking size performs the best?

B) What is the L1/L2/L3 size of the processor you're running on? (`lscpu` or `/proc/cpuinfo` and Google should help)

C) Can you use the information about the cache sizes to predict the best performing block size?

### Hints

There are a few stats that may come in handy:

```
overallMissRate::total
overallAvgMissLatency::total (in Ticks or ps)
simSeconds
```

## Submission

For this assignment you don't need to turn in any code files. The only file you need to submit on gradescope at the designated section, is the pdf file of your report. Please do not forget to specify each question according to the outline when you're submitting your work.
In your report, you're not required to include any data which is not used in your analysis. Only include those that you actually use to justify your answer and make sure they are precisly and cleary specified.

## Grading

The grading breakdown is as follows (they are subject to change, but they show the relative breakdown):

Total points = 100

| #Question       | Points |
|-----------------|--------|
| Question 1.A	  | 5      |
| Question 1.B	  | 10	   |
| Question 2.A	  | 15     |
| Question 2.B    | 15     |
| Question 3.A	  | 10     |
| Question 3.B	  | 15     |
| Question 4.A	  | 15     |
| Question 4.B	  | 5      |
| Question 4.C	  | 10     |


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
