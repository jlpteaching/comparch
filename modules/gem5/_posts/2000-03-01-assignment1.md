---
Author: Jason Lowe-Power
Editor:  Maryam Babaie, Mahyar Samani, Kaustav Goswami
Title: ECS 201A Assignment 1
---

# Assignment 1 -- (100 Points)

Originally from University of Wisconsin-Madison CS/ECE 752 .

Modified for ECS 201A, Winter 2024.

**Due on *{{ site.data.course.dates.gem5_1 }}* 1:59 pm (PST)**: See [Submission](#submission) for details

## Table of Contents

- [Administrivia](#adminstrivia)
- [Introduction](#introduction)
- [Workload](#workload)
- [Experimental setup](#experimental-setup)
- [Analysis and simulation](#analysis-and-simulation)
- [Submission](#submission)
- [Grading](#grading)
- [Academic misconduct reminder](#academic-misconduct-reminder)
- [Hints](#hints)

## Adminstrivia

You should submit your report in **pairs** and in **PDF** format.
Make sure to start early and post any questions you might have on Piazza.
The standard late assignment policy applies.


**<span style="font-size:30px" style="font-weight:bold;">&rarr;</span> Use [classroom: assignment 1]({{site.data.course.201a_assignment1_invitation_link}}) to create an assignment.
You will be asked to <u>join/create</u> an assignment. If your teammate has already created an assignment, please <u>join</u> their assignment instead of creating one assignment otherwise <u>create</u> your assignment and ask your teammate to <u>join</u> the assignment. <span style="font-size:30px" style="font-weight:bold;">&larr;</span>**


## Introduction

In this assignment you are going to:

- see a practical demonstration of the Iron Law of computer architecture,
- measure the performance differences of a single-cycle like processor vs an in-order pipelined processor,
- see how the measured performance scales as CPU clock frequency changes,
- and see the effect of memory bandwidth and latency on measured performance.

You are going to use a matrix multiplication program as the workload for your experiments.
Matrix multiplication is a commonly used kernel in many domains such as linear algebra, machine learning, and fluid dynamics.

## Workload

For this assignment, we are going to use a matrix multiplication program as our workload. The program takes an integer as input that determines the `size` of the square matrices `A`, `B`, and `C`.

```cpp
void multiply(double **A, double **B, double **C, int size)
{
    for (int i = 0; i < size; i++) {
        for (int k = 0; k < size; k++) {
            for (int j = 0; j < size; j++) {
                C[i][j] += A[i][k] * B[k][j];
            }
        }
    }
}
```

You can find the definitions for the workload objects in gem5 under `workloads/workloads.py`.
In this assignment, we will only be using `MatMulWorkload`.
In order to create an object of `MatMulWorkload` you just need to pass matrix size (an integer) `mat_size` to its constructor (`__init__`) function.

**In your configuration choose an appropriate value for `mat_size`**.
 It should be large enough that it makes your workload interesting.
Since changing `mat_size` will influence simulation time, as a guideline, choose a value that results in simulation times less than 10 minutes (hostSeconds < 600).
We found that setting mat_size to 224 will result in a simulation time of around 5 minutes which is a reasonable compromise.

## Experimental setup

For this assignment, we will set up an experiment to see effect of changing a system's component on it performance.
You will need to write configuration scripts using gem5 stdlib that allow you to change the CPU model, CPU and cache frequency, and memory model.
Under the `components` directory, you will find modules that define the different models that you should use in your configuration scripts.

- Board models: You can find all the models you need to use for your CPU (processor) under `components/boards.py`.
You will only be using `HW1RISCVBoard` in this assignment.
- CPU models: You can find all the models you need to use for your CPU (processor) under `components/processors.py`.
- Cache models: You can find all the models you need to use for your cache hierarchy under `components/cache_hierarchies.py`. You will only use `HW1MESITwoLevelCache` in this assignment.
- Memory models: You can find all the models you need to use for your memory under `components/memories.py`.

## Analysis and simulation

Complete the following steps and answer the questions for your report.
Collect data from your simulation runs and use simulator statistics to answer the questions.
Use clear reasoning and visualization to drive your conclusions.
You are allowed to submit your reports in **pairs** and in **PDF** format.

Before starting with simulations, answer the following questions in your report.

1. What metrics should you use to measure the performance of a computer system? Why?
2. Why is it not always possible to use the same metrics for performance to evaluate computer systems?
3. Define the Iron law of processor performance.

### Step I: Changing the CPU model and CPU and cache clock frequency

Before running any simulations try to answer these questions.

1. At the same clock frequency, between a single-cycle CPU (`HW1TimingSimpleCPU`) and an in-order pipelined CPU (`HW1MinorCPU`) which CPU will exhibit better performance? Why?
2. Between a single-cycle CPU (`HW1TimingSimpleCPU`) and an in-order pipelined CPU (`HW1MinorCPU`) CPU which one is going to be more sensitive to changing the clock frequency? Why?

In your configuration script allow for:

- changing the CPU model between `HW1TimingSimpleCPU` and `HW1MinorCPU`
- and changing the clock frequency between `1GHz`, `2GHz`, and `4GHz`

Use `HW1DDR3_1600_8x8` as the memory model.

In your report, answer the same questions after simulation supported with data.
A complete set of simulation data for this step should include **6 configurations** (2 options for CPU model * 3 options for clock frequency).

### Step II: Changing the CPU and memory model

Before running any simulations try to answer these questions:

1. What would be the impact on the overall performance of a computer system if we were to enhance the memory bandwidth and reduce its latency?
2. Which CPU model (between `HW1TimingSimpleCPU` and `HW1MinorCPU`) will benefit more from improving memory performance? Why?

In your configuration allow for:

- changing the CPU model between `HW1TimingSimpleCPU` and `HW1MinorCPU`.
- and changing the memory model between `HW1DDR3_1600_8x8`, `HW1DDR3_2133_8x8`, and `HW1LPDDR3_1600_1x32`.

We can calculate the bandwidth of the memory model.
Let's take an example of HW1DDR3_1600_8x8.
Here we have a DDR3 DRAM device of 1600 MHz and a width of 8 bytes.
The bandwidth of this device can be given by:
- peak BW = 1.6 x 8 = 12.8 GB/s
- Similarly for HWDDR3_2133_8x8, peak BW = 2.133 x 8 = 17.057 GB/s
- For the LPDDR3_1600_1x32, the peak BW = 1.6 x 4 = 6.4 GB/s

You can see the peak bandwidth in the stats.txt.

Latency for memory devices is given by the amount of time it takes to access the data.
The terminology associated with it is called row cycle time (tRC).
tRC is given by row access time (tRAS) and precharge time (tRP).
tRC times are given as:

- HW1DDR3_1600_8x8   --  48.75 ns
- HW1DDR3_2133_8x8   --  46.09 ns
- HW1DDR3_1600_1x32  --  58 ns

The above memory models have the following characteristics

|   memory model          |   frequency   |   width   |   bandwidth   |   latency |
|-------------------------|---------------|-----------|---------------|-----------|
|   `HW1DDR3_1600_8x8`    |  1600 MHz     |      8    |  12.8 GB/s    |  48.75 ns |
|   `HW1DDR3_2133_8x8`    |  2133 MHz     |      8    |  17.057 GB/s  |  46.09 ns |
|   `HW1LPDDR3_1600_1x32` |  1600 MHz     |      4    |  6.4 GB/s     |  58 ns    |

Use `4GHz` as the clock frequency.

**NOTE**: To become familiar with the different memory models you will use in this assignment, please read through the documentation for the different memory models in `components/memories.py`.

In your report, answer the same questions after simulation supported with data.
A complete set of simualtion data for this step should include **6 configurations** (2 options for CPU model * 3 options for memory model).

### Step III: Using different compiler optimizations

In this step, you'll be using different compiler optimizations to run the same
matrix multiplication program.
Compiler optimization flags are options that can be used to improve the
*performance* of the program.
The default version of the matrix multiply program is compiled with the flag `-O2`:

```sh
g++ -o mm mm.cpp -static -O2
```

We have compiled binaries with flags `-O0` and `-O3`.

The former is a binary without any compiler optimizations.
The latter applies most of the optimizations that gcc (or llvm/clang) supports.

This *usually* means that there are fewer instructions in the more optimized version of the program.
Let's look at the following computation: $z = \alpha x + y$.
This operation first multiplies $\alpha$ and $x$, and then adds $y$ to the result.
Assume that x, y and z are all float.
In modern processors, these two steps are usually combined in a single `multiply and accumulate` or `MAC` operation.
If the compiler detects such computations, it'll try to optimize this from two `ld` and one `st` to a MAC operation like `fmadd.d` instruction.
When we look at the assembly, it generates an `fmadd.d rd, rs1, rs2, rs3`.

Let us look at a C++ example.
For the function defined below, we used [godbolt](https://godbolt.org) to generate its RISCV assembly using different compiler optimization flags:
```cpp
float multiply_and_accumulate(float alpha, float x, float y) {
    return alpha * x + y;
}
```
When using `-O0`, we get the following output:
```asm
multiply_and_accumulate(float, float, float):
        addi    sp,sp,-32
        sd      s0,24(sp)
        addi    s0,sp,32
        fsw     fa0,-20(s0)
        fsw     fa1,-24(s0)
        fsw     fa2,-28(s0)
        flw     fa4,-20(s0)
        flw     fa5,-24(s0)
        fmul.s  fa4,fa4,fa5
        flw     fa5,-28(s0)
        fadd.s  fa5,fa4,fa5
        fmv.s   fa0,fa5
        ld      s0,24(sp)
        addi    sp,sp,32
        jr      ra
```
When using `-O1`, we get the following output:
```sam
multiply_and_accumulate(float, float, float):
        fmul.s  fa0,fa0,fa1
        fadd.s  fa0,fa0,fa2
        ret
```
Finally, when using `-O2` or `-O3` for this code, we get the following:
```asm
multiply_and_accumulate(float, float, float):
        fmadd.s fa0,fa0,fa1,fa2
        ret
```

For this experiment, use the `HW1TimingSimpleCPU` and `HW1DDR3_1600_8x8` for this step.

Before running any simulations try to answer this question, which has 4 parts:

1. Which program do you think will perform better?
2. What part of the Iron Law are you optimizing in this step?
3. Do you think if you use CISC ISA, the results will further improve? Why?

In your report, answer the same questions after simulation supported with data.
A complete set of simulation data for this step should include
**two configurations** (one for -O0 and one for -O3)

### Step IV: General questions

Having completed your simulation runs and analyses, answer this final question in your report.

1. If you were to use a different application, do you think your conclusions would change? Why?

## Submission

Your submission is split into two parts. Read the following sections for details on each part.

### Part I: Reproducibility package

As part of your submission, you should include any script/code/file that might be needed to rerun your gem5 experiments.
This may include configuration scripts that define set up the simulation, python/shell/etc. scripts that drive your simulations using your configuration scripts, any document including instruction on how to run your simulations.
You should do this through your assignment's repository.
Ensure that you commit and push your changes from your local repository to the remote repository.
Add clear and relevant commit messages to your commits.
**NOTE**: Any commits/pushes past the assignment deadline will be ignored.

### Part II: Report

As mentioned before, you are allowed to submit your assignments in **pairs**
and in **PDF** format.
You should submit your report on
[gradescope]({{site.data.course.201a_gradescope_lab1_link}}).
In your report answer the questions presented in
[Analysis and simulation](#analysis-and-simulation),
[Analysis and simulation: Step I](#step-i-changing-the-cpu-model-and-cpu-and-cache-clock-frequency),
[Analysis and simulation: Step II](#step-ii-changing-the-cpu-and-memory-model),
[Analysis and simulation: Step III](#step-iii-using-different-compiler-optimizations),
and
[Analysis and simulation: Step IV](#step-iv-general-questions).
Use clear reasoning and visualization to drive your conclusions.

## Grading

Like your submission, your grade is split into two parts.

1. Reproducibility Package (50 points):
    - Instruction and automation to run simulations for different section and
    dump statistics (20 points)
    - Instructions (10 points)
    - Automation (10 points)
    - Configuration scripts and correct simulation setup (30 points): 2.5
    points for each configuration as described in
    [Analysis and simulation: Step I](#step-i-changing-the-cpu-model-and-cpu-and-cache-clock-frequency)
    and
    [Analysis and simulation: Step II](#step-ii-changing-the-cpu-and-memory-model)
2. Report (50 points): See the gradescope page for the complete breakdown for each of the questions.
    - [Analysis and simulation](#analysis-and-simulation) (5 x 2 + 4 points),
    - [Analysis and simulation: Step I](#step-i-changing-the-cpu-model-and-cpu-and-cache-clock-frequency) (5 x 3 points),
    - [Analysis and simulation: Step II](#step-ii-changing-the-cpu-and-memory-model) (5 x 2 points),
    - [Analysis and simulation: Step III](#step-iii-using-different-compiler-optimizations) (2 x 5 + 1 points),
    - [Analysis and simulation: Step IV](#step-iv-general-questions) (5 x 1 points).

## Academic misconduct reminder

You are required to work on this assignment in teams.
You are only allowed to share you scripts and code with your teammate(s).
You may discuss high-level concepts with others in the class but all the work must be completed by your team and your team only.

Remember, DO NOT POST YOUR CODE PUBLICLY ON GITHUB!
Any code found on GitHub that is not the base template you are given will be reported to SJA.
If you want to sidestep this problem entirely, don’t create a public fork and instead create a private repository to store your work.

## Hints

- Start early and ask questions on Piazza and in discussion.
- If you need help, come to office hours for the TA, or post your questions on Piazza.
