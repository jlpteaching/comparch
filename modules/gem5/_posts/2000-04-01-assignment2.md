---
Author: Jason Lowe-Power
Editor:  Maryam Babaie, Mahyar Samani
Title: ECS 201A Assignment 2
---

# Assignment 2 -- Due 11:59 pm (PST) *{{ site.data.course.dates.dino_2 }}* (100 Points)

Originally from University of Wisconsin-Madison CS/ECE 752 .

Modified for ECS 201A, Winter 2023.

**Due on *{{ site.data.course.dates.dino_2 }}* 11:59 pm (PST)**: See [Submission](#submission) for details

## Table of Contents
  - [Introduction](#introduction)
  - [Workload](#Workload)
  - [Experimental setup](#experimental-setup)
  - [Analyzing simulation data](#analyzing-simulation-data)
  - [Submission](#submission)
  - [Grading](#grading)
  - [Academic misconduct reminder](#academic-misconduct-reminder)
  - [Hints](#hints)

## Introduction

You should submit your report in pairs. Make sure to start early and post any questions you might have on Piazza. The standard late assignemt policy applies.

For all the assignment in this course, we are going to be using the recently developed standard library in the gem5 simulator. Please read through [Getting started with gem5 stdlib](). We are going to use github's classroom and codespaces to setup and run our experiments in gem5. This means no compiling gem5.

<!-- TODO: Update this text -->
<!-- The purpose of this assignment is to give you experience with pipelined CPUs. You will your workload with a pipelined in-order CPU to understand how the latency and bandwidth of different parts of pipeline affect performance. You will also be exposed to pseudo-instructions that are used for carrying out functions required by the underlying experiment. This homework is based on exercise 3.6 of CA:AQA 3rd edition (the former textbook for this course) and was developed in part by Jason Lowe-Power et al., then modernized by Matt Sinclair and Jason Lowe-Power. -->

## Workload

For this assignment we are going to use DAXPY as our workload. The DAXPY loop (double precision `aX + Y`) is an often used operation in programs that work with matrices and vectors. The following code implements DAXPY in C++.

```cpp
#include <cstdio>
#include <random>

int main()
{
  const int N = 4096;
  double X[N], Y[N], alpha = 0.5;
  std::random_device rd; std::mt19937 gen(rd());
  std::uniform_real_distribution<> dis(1, 2);
  for (int i = 0; i < N; ++i)
  {
    X[i] = dis(gen);
    Y[i] = dis(gen);
  }

  // Start of daxpy loop
  for (int i = 0; i < N; ++i)
  {
    Y[i] = alpha * X[i] + Y[i];
  }
  // End of daxpy loop

  double sum = 0;
  for (int i = 0; i < N; ++i)
  {
    sum += Y[i];
  }
  printf("%lf\n", sum);
  return 0;
}
```

You can find the definitions for the workload objects in gem5 under `workloads/workloads.py`. In this assignment, we will only be using `DAXPYWorkload`. In order to create an object of `DAXPYWorkload` you just need to call its constructor (`__init__`) function.

<!-- In your report, report the breakup of instructions for different op classes -- and provide a brief analysis of the breakdown. For this, grep for `statExecutedInstType` in the stats.txt file (Note: there are also summary stats directly above the `statExecutedInstType` stats that you may find useful). You should also use the same two-level cache configuration as assignment1. -->

## Experimental setup

In this assignment we are going to measure the impact of different pipeline latencies on the overall performance of the system. As we discussed in the class it is important to consider measured performance as a product of both software and hardware. In this spirit, it might be useful to get a picture of the instruction mix in our workload. You might find this information useful in your analyses. As part of this assignment, you will only modify/change the CPU model. Models for the board, cache hierarchy, and memory will remain a constant in your experiment.

- Board models: You can find all the models you need to use for your CPU (processor) under `components/boards.py`. You will only be using HW1RISCVBoard in this assignment.
- CPU models: You can find all the models you need to use for your CPU (processor) under `components/processors.py`.
- Cache models: You can find all the models you need to use for your cache hierarchy under `components/cache_hierarchies.py`. You will only use HW2MESITwoLevelCache in this assignment.
- Memory models: You can find all the models you need to use for your memory under `components/memories.py`. You will only use HW2DDR3_1600_8x8 in this assignment.

### Step I

`TimingSimpleCPU` is an internal CPU model in gem5's code base that models the execution of non-memory instructions as a single cycle CPU. This CPU model is a useful tool for extracting information on the instruction mix of a program. You can find the definition of `HW2TimingSimpleCPU` which is based on `TimingSimpleCPU` in `components/processors.py`.

Write a configuration script that will simulate the execution of `DAXPYWorkload` on `HW2TimingSimpleCPU`. Make sure to track the simulation outputs for later use. In the statistics output look for `statExecutedInstType`. This statistic represents a distribution of different operation classes executed by the processor.

Hint: Think about how you can apply Amdahl's law in your analyses.

### Step II

For the main part of your experiments, write a configuration script that allows you to simulate `DAXPYWorkload` with `HW2MinorCPU`. Make sure to understand how to instantiate an instance of `HW2MinorCPU`. **NOTE**: Although you can call its constructor function (`__init__`) without any input arguments passed, you will need to set those values for your experimentation. Please make sure to read the documentation for `HW2MinorCPU` and understand what each of the input arguments to `__init__` mean.

`MinorCPU` is one of gem5's internal CPU models that models an in-order pipelined CPU. `HW2MinorCPU` is based on `MinorCPU`. The default pool of functional units for `MinorCPU` includes **two integer units** and **one floating point and SIMD unit**.

Modify your configuration script to allow for changing **floating point issue latency**, and **floating point operation latency**. In this step, measure your simulated performance for different combination of these two latencies. For simplicity's sake, start with an **initial value** of `1 cycle` for **floating point issue latency** and an intial value of `6 cycles` for **floating point operation latency**. Moreover, **assume** you can trade `1 cycle` of **floating point issue latency** with `1 cycle` of **floating point operation latency**. Therefore, in all of your simulation runs, the sum of these two latencies should remain at a constant of `7 cycles`. Here is a table showing all the combinations of these latencies that you need to experiment with.

| # | issue latency | operation latency |
|---|---------------|-------------------|
| 1 | 1             | 6                 |
| 2 | 2             | 5                 |
| 3 | 3             | 4                 |
| 4 | 4             | 3                 |
| 5 | 5             | 2                 |
| 6 | 6             | 1                 |

**NOTE**: Make sure to keep the simulation output for all of your simulation runs for your later analyses.

### Step III

Modify your configuration script to allow for changing **integer issue latency**, **integer operation latency**, **floating point issue latency**, and **floating point operation latency**. For this step lets assume our processor has a very fast **decode** stage that can issue both **integer** and **floating point** instructions in `1 cycle`. Next, lets focus **integer operation latency** and **floating point operation latency**. Now let's assume an intial value of `2 cycles` for **integer operation latency** and an initial value of `4 cycles` for **floating point operation latency**. For your experimentation, suppose you can only reduce one of these latencies by a factor of 2. This means that you can build a processor with an **integer operation latency** of `1 cycle` and a **floating point operation latency** of `4 cycles` or a processor with an **integer operation latency** of `2 cycles` and a **floating point operation latency** of `2 cycles`. For your experimentation, simulate the baseline case and the two possible **improved** cases. Here is a table showing all possible combinations of the latencies that you need to experiment with.

| # | integer issue latency | integer operation latency | floating point issue latency | floating point operation latency |
|---|-----------------------|---------------------------|------------------------------|----------------------------------|
| 1 | 1                     | 2                         | 1                            | 4                                |
| 2 | 1                     | 1                         | 1                            | 4                                |
| 3 | 1                     | 2                         | 1                            | 2                                |

**NOTE**: Make sure to keep the simulation output for all of your simulation runs for your later analyses.

## Analyzing simulation data

After collecting all of the data from your experiments, analyze the statistics your runs generated and write a report. Visualize your simulation data using charts such as line chart and bar charts to drive your conclusions. As part of the assignment repository you can find a starter Jupyter notebook with examples of how your TA extracts statistics and visualizes them. You are allowed to submit your reports in pairs and in PDF/Jupyter notebook format. Make sure to use clear reasoning supported by data (if needed) to answer the following questions in your report.

1. What metric would you choose to compare the performance of different designs in [Step II](#step-ii)? Why?
2. Between the 6 designs for the **floating point** portion of your CPU in [Step II](#step-ii), which one did you find to be the be the best design?
3. What do you think is the reason why your answer to question 2 results in the best performance? Would it make sense to build a **decode** stage that slower than the **execute** stage?
4. What metric would you choose to compare the performance of different designs in [Step III](#step-iii)? Why?
5. Between the 3 designs in [Step III](#step-iii) which design is the best desing to choose?
6. Without using simulation results, can you predict the speed up of each **improved case** over the baseline design?
7. Using simulation results, what is the speed up of each **improved case** in [Step III](#step-iii) over the baseline design?
8. If there are any differences between your answer to questions 6 and 7, what do you think could be the reason?

## Submission

## Grading

## Academic misconduct reminder

## Hints

- Start early and ask questions on Piazza and in discussion.
- If you need help, come to office hours for the TA, or post your questions on Piazza.
