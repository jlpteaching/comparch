---
Author: Jason Lowe-Power
Editor:  Maryam Babaie, Mahyar Samani, Kaustav Goswami
Title: ECS 201A Assignment 2
---

Originally from University of Wisconsin-Madison CS/ECE 752.

Modified for ECS 201A, Winter 2024.

**Due on *{{ site.data.course.dates.gem5_2 }}* 1:59 pm (PST)**: See [Submission](#submission) for details

## Table of Contents

- [Administrivia](#administrivia)
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

Use [classroom: assignment 2]({{site.data.course.201a_assignment1_invitation_link}}) to create an assignment. You will be asked to **join**/**create** an assignment. If your teammate has already created an assignment, please **join** their assignment instead of creating one assignment otherwise **create** your assignment and ask your teammate to **join** the assignment.

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
  const int N = 131072;
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

You can find the definitions for the workload objects in gem5 under `workloads/daxpy_workloads.py`. In this assignment, we will only be using `DAXPYWorkload`. In order to create an object of `DAXPYWorkload` you just need to call its constructor (`__init__`) function.

## Experimental setup

In this assignment we are going to measure the impact of different pipeline latencies on the overall performance of the system.
As we discussed in the class it is important to consider measured performance as a product of both software and hardware.
In this spirit, it might be useful to get a picture of the instruction mix in our workload.
You might find this information useful in later steps of your analysis.
As part of this assignment, you will only modify/change the CPU model.
Models for the board, cache hierarchy, and memory will remain a constant in your experiment.

- Board models: You can find all the models you need to use for your CPU (processor) under `components/boards.py`.
You will only be using `HW2RISCVBoard` in this assignment.
- CPU models: You can find all the models you need to use for your CPU (processor) under `components/processors.py`.
There are a few classes defined in `components/processors.py`.
However, the main classes (models) you will need to use are `HW2TimingSimpleCPU` and `HW2MinorCPU`.
- Cache models: You can find all the models you need to use for your cache hierarchy under `components/cache_hierarchies.py`.
You will only use `HW2MESITwoLevelCache` in this assignment.
- Memory models: You can find all the models you need to use for your memory under `components/memories.py`.
You will only use `HW2DDR4_2400_8x8` in this assignment.
- Clock frequency: You can use a clock frequency of `4 GHz` for all of your simulations.

### Region of Interest (ROI)

In your role as a computer architect, it’s crucial to focus on the code segments that put the most strain on the specific hardware component you’re targeting.
There are usually three segments to a program: (a) initialization, (b) computation, and (c) verification.
As you might have guessed, segment (b) is the important section that you need to study.
In the DAXPY code above, our ROI is the DAXPY loop.
```cpp
  // Start of daxpy loop
  for (int i = 0; i < N; ++i)
  {
    Y[i] = alpha * X[i] + Y[i];
  }
  // End of daxpy loop
```
In gem5, you can annotate this region with gem5-specific instruction.
In the `workloads/daxpy/daxpy.cpp`, the code is annotated as:
```cpp
#ifdef GEM5
  m5_work_begin(0,0);
#endif

  // Start of daxpy loop
  for (int i = 0; i < N; ++i)
  {
    Y[i] = alpha * X[i] + Y[i];
  }
  // End of daxpy loop

#ifdef GEM5
  m5_work_end(0,0);
#endif
```
To compile this program, you need to include the `gem5/m5ops.h` header file.
In the stats file generated after the simulation, you will only have statistics within the defined ROI.

The compiled program `daxpy-gem5` and it's assembly are already generated in the `workloads/daxpy/`.

### **IMPORTANT NOTE**

In your configuration scripts, make sure to import `exit_event_handler` using the command below.

```python
from workloads.roi_manager import exit_event_handler
```

You will have to pass `exit_event_handler` as a keyword argument named `on_exit_event` when creating a `simulator` object. Use the *template* below to create a simulator object.

```python
simulator = Simulator(board={name of your board},
                      full_system=False,
                      on_exit_event=exit_event_handler)
```

## Analysis and simulation

Complete the following steps and answer the questions for your report.
Collect data from your simulation runs and use simulator statistics to answer the questions.
Use clear reasoning and visualization to drive your conclusions.
You are allowed to submit your reports in **pairs** and in **PDF** format.

### Step 0

Before starting simulation and analysis, you should be able to identify the
ROI of a  program.

1. For the DAXPY's assembly code, identify the ROI. In your report, copy the assembly code segment corresponding to `m5_work_begin` and `m5_work_end`.  

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
Use `HelloWorldWorkload` from `workloads/hello_world_workload.py` as a second program to compare instruction mixes.
A complete set of simulation data for this step should include **two configuration** (one for `DAXPYWorkload` and one for `HelloWorldWorkload`).

### Step II

For this step, write a configuration script that allows you to simulate `DAXPYWorkload` with `HW2MinorCPU`.
Make sure to understand how to instantiate an instance of `HW2MinorCPU`.
**NOTE**: Although you can call its constructor function (`__init__`) without any input arguments passed, you will need to set those values for your experimentation.
Please make sure to read the documentation for `HW2MinorCPU` and understand what each of the input arguments to `__init__` mean.

`MinorCPU` is one of gem5's internal CPU models that models an in-order pipelined CPU.
`HW2MinorCPU` is based on `MinorCPU`.
The default pool of functional units for `MinorCPU` includes **two integer units** and **one floating point and SIMD unit**.

Modify your configuration script to allow for changing **issue latency**, and **floating point operation latency**.
For your reference, **issue latency** measures the number of cycles between injection two consecutive instructions into the pipeline.
An **issue latency** of `4 cycles` means that an instruction is injected to the pipeline, every *4 cycles*.
On the other hand, **floating point operation latency** refers to the number of cycles it takes to complete the execution of a **floating point instruction**.
In this step, measure your simulated performance for different combination of these two latencies.
For simplicity's sake, start with an **initial value** of `4 cycles` for **issue latency** and an intial value of `4 cycles` for **floating point operation latency**.
Moreover, **assume** you can trade **issue latency** with **floating point operation latency**.
In addition, **assume** that the product of **issue latency** and **floating point operation latency** will always remain at a constant of `8`.
For your simulations, evaulate the performance of the configurations shown below.

| # | issue latency | floating point operation latency |
|---|---------------|----------------------------------|
| 1 | 4             | 2                                |
| 2 | 2             | 4                                |
| 3 | 8             | 1                                |

**NOTE**: Make sure to keep track of your simulation outputs for all of your simulation runs for your later analyses.

In your report, answer the following questions after simulation supported with data.
A complete set of simulation data for this step should include **three configurations** (three possible combinations of **issue latency** and **floating point operation latency**).

1. Between the 3 designs, which one did you find to be the be the best design?
2. Why do you think your chosen design in question 1 results in the best
performance?
Can you reason about why you would prefer optimizing one of the latencies over
the other?

### Step III

For this step, modify your configuration script to allow for changing **integer operation latency** and **floating point operation latency**.
Let's assume our processor has a very fast **decode** stage that can issue both **integer** and **floating point** instructions in `1 cycle`.
Next, let's focus on **integer operation latency** and **floating point operation latency**.
Let's assume an intial value of `6 cycles` for **integer operation latency** and an initial value of `12 cycles` for **floating point operation latency**.
For your experimentation, suppose you can only reduce one of these latencies by
a factor of 2.
This means that you can build a processor with an **integer operation latency** of `3 cycles` and a **floating point operation latency** of `12 cycles` or a processor with an **integer operation latency** of `6 cycles` and a **floating point operation latency** of `6 cycles`.
For your experimentation, simulate the baseline case and the two possible **improved** cases.
Here is a table showing all possible combinations of the latencies that you need to experiment with.

| # | integer issue latency | integer operation latency | floating point issue latency | floating point operation latency |
|---|-----------------------|---------------------------|------------------------------|----------------------------------|
| 1 | 1                     | 6                         | 1                            | 12                               |
| 2 | 1                     | 3                         | 1                            | 12                               |
| 3 | 1                     | 6                         | 1                            | 6                                |

In your report answer the following questions.

1. Use Amdahl's law and the information you gathered from [Step I](#step-i) to predict the speed up of each **improved** case over the **baseline**.
Which design would you choose?
**NOTE**: The only simulation result you can use to answer this question is the data you gathered from [Step I](#step-i).
2. Using simulation results, what is the speed up of each **improved case** over the baseline design?
3. If there are any differences between your answer to questions 1 and 2, what do you think could be the reason?

**Hints**:

- Take a look at the assembly code for the `DAXPY` loop below (you can also find the complete assembly for it under `worklaods/daxpy/daxpy-gem5-asm`). Can you point out some dependencies between the instructions? Do you think only looking at the instruction mix gathered from [Step I](#step-i) provided enough information to apply Amdahl's law?
- Think about the other stages of the pipeline, in this question we have only focused on **decode** and **execute**.

```asm
.L35:
# daxpy.cpp:27:     Y[i] = alpha * X[i] + Y[i];
	fld	fa4,0(a5)	# MEM[(double *)_56], MEM[(double *)_56]
	fld	fa5,0(s2)	# MEM[(double *)_49], MEM[(double *)_49]
# daxpy.cpp:25:   for (int i = 0; i < N; ++i)
	addi	a5,a5,8	#, ivtmp.133, ivtmp.133
	addi	s2,s2,8	#, ivtmp.132, ivtmp.132
# daxpy.cpp:27:     Y[i] = alpha * X[i] + Y[i];
	fmadd.d	fa5,fa5,fa3,fa4	# _5, MEM[(double *)_49], tmp181, MEM[(double *)_56]
# daxpy.cpp:27:     Y[i] = alpha * X[i] + Y[i];
	fsd	fa5,-8(a5)	# _5, MEM[(double *)_56]
# daxpy.cpp:25:   for (int i = 0; i < N; ++i)
	bne	s1,a5,.L35	#, _14, ivtmp.133,
```

**NOTE**: Make sure to keep the simulation output for all of your simulation runs for your later analyses.

## Submission

As mentioned before, you are allowed to submit your assignments in **pairs** and in **PDF** format.
You should submit your report on
[gradescope](https://www.gradescope.com/courses/487868).
In your report answer the questions presented in [Analysis and simulation](#analysis-and-simulation), [Analysis and simulation: Step 0](#step-0),
[Analysis and simulation: Step I](#step-i),
[Analysis and simulation: Step II](#step-ii), and
[Analysis and simulation: Step III](#step-iii).
Use clear reasoning and visualization to drive your conclusions.
Submit all your code through your assignment repository.
Please make sure to include code/scripts for the following.

- `Instruction.md`: should include instruction on how to run your simulations.
- Automation: code/scripts to run your simulations.
- Configuration: python file configuring the systems you need to simulate.

## Grading

Like your submission, your grade is split into two parts.

1. Reproducibility Package (50 points):
    1. Instruction and automation to run simulations for different section and
    dump statistics (20 points)
        - Instructions (10 points)
        - Automation (10 points)
    2. Configuration scripts and correct simulation setup (30 points): 3 points
    for each configuration as described in
    [Analysis and simulation: Step I](#step-i),
    [Analysis and simulation: Step II](#step-ii),
    and [Analysis and simulation: Step III](#step-iii)
2. Report (50 points): 1 point for [Analysis and simulation: Step 0](#step-0),
7 points for each question presented in
[Analysis and simulation: Step I](#step-i),
[Analysis and simulation: Step II](#step-ii),
[Analysis and simulation: Step III](#step-iii)

## Academic misconduct reminder

You are required to work on this assignment in teams. You are only allowed to share you scripts and code with your teammate(s). You may discuss high level concepts with others in the class but all the work must be completed by your team and your team only.

Remember, DO NOT POST YOUR CODE PUBLICLY ON GITHUB! Any code found on GitHub that is not the base template you are given will be reported to SJA. If you want to sidestep this problem entirely, don’t create a public fork and instead create a private repository to store your work.

## Hints

- Start early and ask questions on Piazza and in discussion.
- If you need help, come to office hours for the TA, or post your questions on Piazza.
