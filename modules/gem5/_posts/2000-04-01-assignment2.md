---
Author: Jason Lowe-Power
Editor:  Maryam Babaie, Mahyar Samani
Title: ECS 201A Assignment 2
---

Originally from University of Wisconsin-Madison CS/ECE 752.

Modified for ECS 201A, Winter 2023.

**Due on *{{ site.data.course.dates.gem5_2 }}* 11:59 pm (PST)**: See [Submission](#submission) for details

## Table of Contents

- [Introduction](#introduction)
- [Workload](#workload)
- [Experimental setup](#experimental-setup)
- [Analysis and simulation](#analysis-and-simulation)
- [Submission](#submission)
- [Grading](#grading)
- [Academic misconduct reminder](#academic-misconduct-reminder)
- [Hints](#hints)

## Administrivia

You should submit your report in pairs. Make sure to start early and post any questions you might have on Piazza. The standard late assignemt policy applies.

Use [classroom: assignment 2]() to create an assignment. You will be asked to **join**/**create** an assignment. If your teammate has already created an assignment, please **join** their assignment instead of creating one assignment otherwise **create** your assignment and ask your teammate to **join** the assignment.

## Introduction

In this assignment, you are going to:

- Learn how you can use gem5 to profile programs,
- Evaluate the performance of different configurations of a pipeline,
- Use Amdahl's law in practice.

This homework is based on exercise 3.6 of [CA:AQA 3rd edition](https://www.google.com/books/edition/Computer_Architecture/XX69oNsazH4C) (the former textbook for this course) and was developed in part by Jason Lowe-Power et al., then modernized by Matt Sinclair and Jason Lowe-Power.

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

## Experimental setup

In this assignment we are going to measure the impact of different pipeline latencies on the overall performance of the system.
As we discussed in the class it is important to consider measured performance as a product of both software and hardware.
In this spirit, it might be useful to get a picture of the instruction mix in our workload.
You might find this information useful in later steps of your analysis.
As part of this assignment, you will only modify/change the CPU model.
Models for the board, cache hierarchy, and memory will remain a constant in your experiment.

- Board models: You can find all the models you need to use for your CPU (processor) under `components/boards.py`. You will only be using `HW2RISCVBoard` in this assignment.
- CPU models: You can find all the models you need to use for your CPU (processor) under `components/processors.py`.
- Cache models: You can find all the models you need to use for your cache hierarchy under `components/cache_hierarchies.py`. You will only use `HW2MESITwoLevelCache` in this assignment.
- Memory models: You can find all the models you need to use for your memory under `components/memories.py`. You will only use `HW2DDR3_1600_8x8` in this assignment.

## Analysis and simulation

Complete the following steps and answer the questions for your report.
Collect data from your simulation runs and use simulator statistics to answer the questions.
Use clear reasoning and visualization to drive your conclusions.
You are allowed to submit your reports in **pairs** and in **PDF** format.

### Step I

Before running any simulations try to answer these questions.
Try to make an educated guess.

1. If you were to divide instructions in a program into the three categories of `integer`, `floating point`, and `memory` instructions, do you think each category would constitute equal parts of a program?
2. Do you think different programs will have a different mix of these categories? Why?

**Recommended Reading**: I recommend you read up on these concept(s): **arithmetic intensity**, **roofline model**.

`TimingSimpleCPU` is an internal CPU model in gem5's code base that models the execution of non-memory instructions as a single cycle CPU.
This CPU model is a useful tool for extracting information on the instruction mix of a program.
You can find the definition of `HW2TimingSimpleCPU` which is based on `TimingSimpleCPU` in `components/processors.py`.

Write a configuration script that will simulate the execution of `DAXPYWorkload` on `HW2TimingSimpleCPU`.
Make sure to track the simulation outputs for later use. In the statistics output look for `statExecutedInstType`.
This statistic represents a distribution of different operation classes executed by the processor.

In your report, answer the same questions after simulation supported with data.
A complete set of simulation data for this step should include **1 configuration**.

### Step II

For this step, write a configuration script that allows you to simulate `DAXPYWorkload` with `HW2MinorCPU`.
Make sure to understand how to instantiate an instance of `HW2MinorCPU`.
**NOTE**: Although you can call its constructor function (`__init__`) without any input arguments passed, you will need to set those values for your experimentation.
Please make sure to read the documentation for `HW2MinorCPU` and understand what each of the input arguments to `__init__` mean.

`MinorCPU` is one of gem5's internal CPU models that models an in-order pipelined CPU.
`HW2MinorCPU` is based on `MinorCPU`. The default pool of functional units for `MinorCPU` includes **two integer units** and **one floating point and SIMD unit**.

Modify your configuration script to allow for changing **floating point issue latency**, and **floating point operation latency**.
In this step, measure your simulated performance for different combination of these two latencies.
For simplicity's sake, start with an **initial value** of `1 cycle` for **floating point issue latency** and an intial value of `6 cycles` for **floating point operation latency**.
Moreover, **assume** you can trade `1 cycle` of **floating point issue latency** with `1 cycle` of **floating point operation latency**.
Therefore, in all of your simulation runs, the sum of these two latencies should remain at a constant of `7 cycles`.
Here is a table showing all the combinations of these latencies that you need to experiment with.

| # | floating point issue latency | floating point operation latency |
|---|------------------------------|----------------------------------|
| 1 | 1                            | 6                                |
| 2 | 2                            | 5                                |
| 3 | 3                            | 4                                |
| 4 | 4                            | 3                                |
| 5 | 5                            | 2                                |
| 6 | 6                            | 1                                |

**NOTE**: Make sure to keep the simulation output for all of your simulation runs for your later analyses.

In your report, answer the same questions after simulation supported with data.
A complete set of simulation data for this step should include **6 configurations** (6 possible combination of **issue** and **operation** latency).

1. Between the 6 designs for the **floating point** unit, which one did you find to be the be the best design?
2. What do you think is the reason why your answer to question 2 results in the best performance? Would it make sense to build a **decode** stage that slower than the **execute** stage?

### Step III

Modify your configuration script to allow for changing **integer issue latency**, **integer operation latency**, **floating point issue latency**, and **floating point operation latency**.
For this step lets assume our processor has a very fast **decode** stage that can issue both **integer** and **floating point** instructions in `1 cycle`.
Next, let's focus **integer operation latency** and **floating point operation latency**.
Let's assume an intial value of `2 cycles` for **integer operation latency** and an initial value of `4 cycles` for **floating point operation latency**.
For your experimentation, suppose you can only reduce one of these latencies by a factor of 2.
This means that you can build a processor with an **integer operation latency** of `1 cycle` and a **floating point operation latency** of `4 cycles` or a processor with an **integer operation latency** of `2 cycles` and a **floating point operation latency** of `2 cycles`.
For your experimentation, simulate the baseline case and the two possible **improved** cases.
Here is a table showing all possible combinations of the latencies that you need to experiment with.

| # | integer issue latency | integer operation latency | floating point issue latency | floating point operation latency |
|---|-----------------------|---------------------------|------------------------------|----------------------------------|
| 1 | 1                     | 2                         | 1                            | 4                                |
| 2 | 1                     | 1                         | 1                            | 4                                |
| 3 | 1                     | 2                         | 1                            | 2                                |

In your report answer the following questions.

1. Use Amdahl's law and the information you gathered from [Step I](#step-i) to predict the speed up of each **improved** case over the **baseline**. Which design would you choose? **NOTE**: The only simulation result you can use to answer this question is the data you gathered from [Step I](#step-i)[Step I](#step-i).
2. Using simulation results, what is the speed up of each **improved case** over the baseline design?
3. If there are any differences between your answer to questions 1 and 2, what do you think could be the reason? **Hint**: A computer program has instructions other that `integer` and `floating point` instructions. In addition, a computer has components other than the `CPU`.

**NOTE**: Make sure to keep the simulation output for all of your simulation runs for your later analyses.

### Step IV: General questions

In your report, answer the following question.

1. What components in a computer system influence the overall performance of the system? Are they all hardware related? **Hint**: Think about all your assignments so far. Think about what Patterson and Hennessey talked about in their Turing Talk.

## Submission

As mentioned before, you are allowed to submit your assignments in **pairs** and in **PDF** format.
You should submit your report on [gradescope](https://www.gradescope.com/courses/487868).
In your report answer the questions presented in [Analysis and simulation](#analysis-and-simulation), [Analysis and simulation: Step I](#step-i), [Analysis and simulation: Step II](#step-ii), [Analysis and simulation: Step III](#step-iii), and [Analysis and simulation: Step IV](#step-iv-general-questions).
Use clear reasoning and visualization to drive your conclusions.

## Grading

Like your submission, your grade is split into two parts.

1. Reproducibility Package (50 points):
    a. Instruction and automation to run simulations for different section and dump statistics (20 points)
        - Instructions (10 points)
        - Automation (10 points)
    b. Configuration scripts and correct simulation setup (30 points): 3 points for each configuration as described in [Analysis and simulation: Step I](#step-i), [Analysis and simulation: Step II](#step-ii), and [Analysis and simulation: Step III](#step-iii)
2. Report (50 points): 7 points for each question presented in [Analysis and simulation: Step I](#step-i), [Analysis and simulation: Step II](#step-ii), [Analysis and simulation: Step III](#step-iii), [Analysis and simulation: Step IV](#step-iv-general-questions)

## Academic misconduct reminder

You are required to work on this assignment in teams. You are only allowed to share you scripts and code with your teammate(s). You may discuss high level concepts with others in the class but all the work must be completed by your team and your team only.

Remember, DO NOT POST YOUR CODE PUBLICLY ON GITHUB! Any code found on GitHub that is not the base template you are given will be reported to SJA. If you want to sidestep this problem entirely, don’t create a public fork and instead create a private repository to store your work.

## Hints

- Start early and ask questions on Piazza and in discussion.
- If you need help, come to office hours for the TA, or post your questions on Piazza.
