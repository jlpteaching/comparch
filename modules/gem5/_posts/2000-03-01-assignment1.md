---
Author: Jason Lowe-Power
Editor:  Maryam Babaie, Mahyar Samani
Title: ECS 201A Assignment 1
---

# Assignment 1 -- Due 11:59 pm (PST) *{{ site.data.course.dates.dino_1 }}* (40 Points)

Originally from University of Wisconsin-Madison CS/ECE 752 .

Modified for ECS 201A, Winter 2023.

**Due on *{{ site.data.course.dates.dino_1 }}* 11:59 pm (PST)**: See [Submission](#submission) for details

## Table of Contents
  - [Introduction](#introduction)
  - [Workload](#workload)
  - [Experimental setup](#experimental-setup)
  - [Analyzing simulation data](#analyzing-simulation-data)
  - [Submission](#submission)
  - [Grading](#grading)
  - [Academic misconduct reminder](#academic-misconduct-reminder)
  - [Hints](#hints)

## Introduction

You should submit your report in pairs. Make sure to start early and post any questions you might have on Piazza. The standard late assignemt policy applies.

For all the assignment in this course, we are going to be using the recently developed standard library in the gem5 simulator. Please read through [Getting started with gem5 stdlib](). We are going to use github's classroom and codespaces to setup and run our experiments in gem5. This means no compiling gem5.

## Workload

For this assignment we are going to use a matrix multiplication program as our workload. The program takes and integer as input that determines the `size` of the square matrices `A`, `B`, and `C`.

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

You can find the definitions for the workload objects in gem5 under `workloads/workloads.py`. In this assignment, we will only be using `MatMulWorkload`. In order to create an object of `MatMulWorkload` you just need to pass `mat_size` to its constructor (`__init__`) function. **In your configuration choose an appropriate value for `mat_size`**. It should be large enough that it makes your workload interesting. Since changing `mat_size` will influence simulation time, as a guideline, choose a value that results in simulation times less than 10 minutes (hostSeconds < 600). We found that setting mat_size to 256 will result in a simulation time of around 7 minutes which is a reasonable compromise.

## Experimental setup

For this assignment, we will set up an experiment to see effect of changing a system's component on it performance. You will need to write a configuration script in gem5 stdlib that allows you to change the CPU model, CPU and cache frequency, and memory model.
Under the `components` directory, you will find modules that define the different models that you should use in your configuration scritps.

- Board models: You can find all the models you need to use for your CPU (processor) under `components/boards.py`. You will only be using HW1RISCVBoard in this assignment.
- CPU models: You can find all the models you need to use for your CPU (processor) under `components/processors.py`.
- Cache models: You can find all the models you need to use for your cache hierarchy under `components/cache_hierarchies.py`. You will only use HW1MESITwoLevelCache in this assignment.
- Memory models: You can find all the models you need to use for your memory under `components/memories.py`.

### Step I: Changing the CPU model

In your config script change the CPU model from HW1TimingSimpleCPU to HW1MinorCPU.

Hint: you may want to read the name of the CPU model as an input from the command line.

### Step II: Varying CPU and cache clock frequency

In your config script change the CPU and cache clock frequncy from `1GHz` to `2GHz` to `4GHz`.

Hint: again, you may want to read the clock frequency as an input from the command line.

### Step III: Varying memory model

In your configuration change the memory model to any of the models listed below.

- HW1DDR3_1600_8x8, which models DDR3 memory.
- HW1DDR3_2133_8x8, which models DDR3 with a faster clock.
- HW1LPDDR3_1600_1x32, which models LPDDR3, a low-power DRAM often found in mobile devices.

Hint: again, you may want to read the name of the memory model as an input from the command line.

### Step IV: Gathering performance data

To gather your performance data, run your `MatMulWorkload` with all the possible combinations of CPU model, CPU and cache clock frequency, and memory model. Make sure to dump the simulator output for each combination to a unique destination so that you can compare and contrast the data later on. In total you need to gather 18 simulation statistics outputs for your data (2 (CPU models) * 3 (clock frequencies) * 3 (memory models)).

## Analyzing simulation data

After collecting all of the data from your experiments, analyze the statistics your runs generated and write a report. Visualize your simulation data using charts such as line chart and bar charts to drive your conclusions. As part of the assignment repository you can find a starter Jupyter notebook with examples of how your TA extracts statistics and visualizes them. You are allowed to submit your reports in pairs and in PDF/Jupyter notebook format. Make sure to use clear reasoning supported by data (if needed) to answer the following questions in your report.

1. What metric should you use to compare the performance between different system configurations? Why is this the appropriate metric?
2. Which CPU model is more sensitive to changing the CPU frequency? Why?
3. Which CPU model is more sensitive to changing the memory technology? Why?
4. Is your measured performance more sensitive to the CPU model, the memory technology, or CPU frequency? Why?
5. If you were to use a different application, do you think your conclusions would change? Why?

## Submission

## Grading

## Academic misconduct reminder
<!--
You are to work on this project **individually**.
You may discuss *high level concepts* with one another (e.g., talking about the sieve algorithm), but all work must be completed on your own.

**Remember, DO NOT POST YOUR CODE PUBLICLY ON GITHUB!**
Any code found on GitHub that is not the base template you are given will be reported to SJA.
If you want to sidestep this problem entirely, don't create a public fork and instead create a private repository to store your work. -->

## Hints

- Start early and ask questions on Piazza and in discussion.
- If you need help, come to office hours for the TA, or post your questions on Piazza.
