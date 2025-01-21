---
Author: Jason Lowe-Power
Editor:  Maryam Babaie, Mahyar Samani, Kaustav Goswami
Title: Effects of Pipelining
---

Originally from University of Wisconsin-Madison CS/ECE 752.

Modified for ECS 201A, Winter 2025.

**Due on 1/27 11:59 pm (PST)**: See [Submission](#submission) for details

GitHub Classroom link for 154B: [{{ site.data.course.154b_assignment2_invitation_link }}]({{ site.data.course.154b_assignment2_invitation_link }})
GitHub Classroom link for 201A: [{{ site.data.course.201a_assignment2_invitation_link }}]({{ site.data.course.201a_assignment2_invitation_link }})

## Table of Contents

- [Introduction](#introduction)
- [Research question](#research-question)
- [Workload](#workload)
- [Experimental setup](#experimental-setup)
- [Analysis and simulation](#analysis-and-simulation)
- [Submission](#submission)
- [Grading](#grading)
- [Academic misconduct reminder](#academic-misconduct-reminder)

## Introduction

In this assignment, you are going to:

- Learn how you can use gem5 to profile programs,
- Evaluate the performance of different configurations of a pipeline,
- Use Amdahl's law in practice.

This homework is based on exercise 3.6 of [CA:AQA 3rd edition](https://www.google.com/books/edition/Computer_Architecture/XX69oNsazH4C) (the former textbook for this course) and was developed in part by Jason Lowe-Power et al., then modernized by Matt Sinclair and Jason Lowe-Power.

## Research question

**How do different pipeline latencies affect the performance of a computer system?**

To answer this question, you will need to simulate a simple workload and measure the performance of the system under different pipeline latencies.

You will be changing the latency and pipeline stages for integer and floating point operations.

We would like to be able to answer the following questions:

1. Does changing the latency of the integer or floating point operations have a bigger impact on the performance of the system?
2. Are these changes the main factor in the performance of the system for the DAXPY workload?

## Workload

For this assignment we are going to use DAXPY as our workload.
The DAXPY loop (double precision `aX + Y`) is an often used operation in programs that work with matrices and vectors.
The following code implements DAXPY in C++.

```cpp
#include <cstdio>
#include <random>

int main()
{
  const int N = 196608;
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

The code for this application is provided in the `workloads/daxpy/` directory.

There are two workloads available for you to use in this assignment using `obtain_resources`:

- "daxpy_riscv_run": The base daxpy code without annotations
- "daxpy_riscv_gem5_run": The daxpy code with gem5's ROI annotations. See [Region of Interest (ROI)](#region-of-interest-roi) for more information.

## Experimental setup

In this assignment we are going to measure the impact of different pipeline latencies on the overall performance of the system.
As we discussed in the class it is important to consider measured performance as a product of both software and hardware.
In this spirit, it might be useful to get a picture of the instruction mix in our workload.
You might find this information useful in later steps of your analysis.
As part of this assignment, you will only modify/change the CPU model.
Models for the board, cache hierarchy, and memory will remain a constant in your experiment.

- Board models: You will only be using `RISCVBoard` in this assignment.
- CPU models: You will use `SingleCycleCPU` and `PipelinedCPU`.
  Note that the pipelined CPU provide has parameters that you can change.
  See the code in `components/processors.py` for more information.
- Cache models: You will only use `MESITwoLevelCache` in this assignment.
- Memory models: You will only use `DDR4` in this assignment.
- Clock frequency: Use a clock frequency of `1 GHz` for all of your simulations.

### Region of Interest (ROI)

In your role as a computer architect, it's crucial to focus on the code segments that put the most strain on the specific hardware component you're targeting.
There are usually three segments to a program: (a) initialization, (b) computation, and (c) verification.
As you might have guessed, segment (b) is the important section that you need to study.
In the DAXPY code above, our region of interest (ROI) is the DAXPY loop.

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
You can use `make` to compile the program yourself (though this is not necessary).

There are also makefile targets to generate the assembly code for the program, which may be useful in your analysis.

## Analysis and simulation

Complete the following steps and answer the questions for your report.
Collect data from your simulation runs and use simulator statistics to answer the questions.
Use clear reasoning and visualization to drive your conclusions.

### Step I: Write down your hypotheses

Before starting simulation and analysis, you should be able to identify the ROI of a  program.
For your reference, **issue latency** measures the number of cycles between injection two consecutive instructions into the pipeline.
An **issue latency** of `4 cycles` means that an instruction is injected to the pipeline, once every *4 cycles*.
On the other hand, **floating point operation latency** refers to the number of cycles it takes to complete the execution of a **floating point instruction**.
A **floating point operation latency** of `8 cycles` means that it will take `8 cycles` for the instruction to be executed once injected into the pipeline.

1. For the DAXPY's assembly code, identify the ROI. In your report, copy the assembly code segment corresponding to the code between `m5_work_begin` and `m5_work_end`.
2. For the ROI of this workload, what percentage of instructions do you think will be integer, floating point, and memory operations? Explain your reasoning.
3. As a baseline, use a pipelined system with an integer operation latency of 1 cycle, a floating point operation latency of 6 cycles, and an issue latency of 1 cycle.
Estimate how the performance will change under the following conditions:
  - If the latency of integer operations are increased from 1 to 6 cycles, but the system is pipelined.
  - If the latency of floating point operations are increased from 6 to 12 cycles, but the system is pipelined.
  - If the issue latency is increased from 1 to 2 cycles, but the operation latency is unchanged (1 cycle for integer and 6 cycles for floating point operations).

Hint: For #3, you should use the Iron Law and an estimated instruction mix.

Now, let's design an experiment to test this hypothesis.

### Step II: Get preliminary data on the instruction mix

Here, we will validate (or invalidate) our hypotheses from [Step I](#step-i-write-down-your-hypotheses-and-experimental-setup) by running simulations.

The `TimingSimpleCPU` is an internal CPU model in gem5's code base that models the execution of non-memory instructions as a single cycle CPU.
This CPU model is a useful tool for extracting information on the instruction mix of a program.
You can find the definition of `SingleCycleCPU` which is based on `TimingSimpleCPU` in `components/`.

Write a configuration script that will simulate the execution of DAXPY with `SingleCycleCPU`.
Make sure to track the simulation outputs for later use.
In the statistics output, look for `statExecutedInstType`.
This statistic represents a distribution of different operation classes executed by the processor.

Now, that we have the instruction mix, let's answer the following questions (the same as above).

1. For the ROI of this workload, what percentage of instructions are integer, floating point, and memory operations? Explain your reasoning.
2. Estimate how the performance will change under the following conditions:
  - If the latency of integer operations are increased from 1 to 6 cycles, but the system is pipelined.
  - If the latency of floating point operations are increased from 6 to 12 cycles, but the system is pipelined.
  - If the issue latency is increased from 1 to 2 cycles, but the operation latency is unchanged (1 cycle for integer and 6 cycles for floating point operations).

### Step II: Developing and running the experiments

For this step, write a configuration script that allows you to simulate DAXPY with the `PipelineCPU`.
Make sure to understand how to instantiate an instance of `PipelineCPU`.
**NOTE**: Although you can call its constructor function (`__init__`) without any input arguments passed, you will need to set those values for your experimentation.
Please make sure to read the documentation for `PipelineCPU` and understand what each of the input arguments to the constructor mean.

`MinorCPU` is one of gem5's internal CPU models that models an in-order pipelined CPU.
`PipelineCPU` is based on `MinorCPU`.
The default pool of functional units for `MinorCPU` includes **two integer units** and **one floating point and SIMD unit**.

Modify your configuration script to allow for changing **issue latency** (which is the same for integer and floating point), **integer latency**, and **floating point operation latency**.


Design three experiments to test your hypothesis about the performance impact of increasing the integer operation latency, floating point operation latency, and issue latency.

1. For each experiment, what is changing in the system compared to the baseline when:
  a. Changing the latency of integer operations.
  b. Changing the latency of floating point operations.
  c. Changing the issue latency.
2. What is the performance change for each condition when:
  a. Changing the latency of integer operations.
  b. Changing the latency of floating point operations.
  c. Changing the issue latency.

### Research question:

Use the data from the experiments that you ran in [Step II](#step-ii-developing-and-running-the-experiments) to answer the following questions.

1. Does changing the latency of the integer operations, floating point operations, or the issue latency have a bigger impact on the performance of the system?
2. Are these changes the main factor in the performance of the system for the DAXPY workload? If not, what other factors might be affecting the performance of the system?

**Hints**:

- Take a look at the assembly code for the `DAXPY` loop below (you can generate the complete assembly for it under `workloads/daxpy` with the makefile).
Can you find some dependencies between the instructions?
Do you think only looking at the instruction mix gathered from [Step I](#step-i-write-down-your-hypotheses) provided enough information to apply instruction mix and the Iron Law?
- Think about the other stages of the pipeline, in this question we have only focused on **decode** and **execute**.

```asm
  call  m5_work_begin@plt  #
# daxpy.cpp:27:     Y[i%N] = alpha * X[i%N] + Y[i%N];
  li  a3,3153920    # tmp338,
  fld  fa3,.LC4,a5  # tmp291,, tmp304
  li  a1,-3145728   # tmp257,
  addi  a5,a3,1824  #, tmp337, tmp338
  add  a5,a5,a1     # tmp257, tmp337, tmp337
  ...
# daxpy.cpp:32:   m5_work_end(0,0);
```

**NOTE**: Make sure to keep the simulation output for all of your simulation runs for your later analyses.

### Next steps (required 201A, extra credit 154B):

Answer the following questions in your report.
Include information on how you designed the experiment, what you measured, and the analyzed the data.

1. Change the clock to be higher (e.g., 4 GHz). How does a higher clock affect the answers to the two research questions?

## Submission

You will submit this assignment via GitHub Classroom.

1. Accept the assignment by clicking on the link provided in the announcement.
2. Create a Codespace for the assignment on your repository.
3. Fill out the `questions.md` file.
4. Commit your changes.

Make sure you include both your runscript, an explanation of how to use your script, and the questions to the questions in the `questions.md` file.

### Explanation of how to use your script

Include a detailed explanation of how to use your script and how you use your script to generate your answers (this will be more applicable in future assignments).
Make sure that all paths are relative to this base directory.
The code included in the "Example command to run the script" section should be able to be copied and pasted into a terminal and run without modification.

- You should include a sentence or two which describes what the script (or scripts) do under "Explanation of the script" in `questions.md`.
- You should include the path to the script under "Script to run" in `questions.md`.
- You should include any parameters that need to be passed to the script under "Parameters to script (if any)" in `questions.md`.
- You should include each command used to gather data under "Command used to gather data" in `questions.md`.
  - Make sure this can by copy-pasted and run in your codespace without modification.
  - If you need other files to run your script, make sure to include those files when you commit your changes.

## Grading

- **25 points** gem5 runscript and explanation of how to use your script
- **45 points** for the questions in the report
- **30 points** for the research question
- **25 points** for the next steps

## Academic misconduct reminder

You are required to work on this assignment individually.
You may discuss high level concepts with others in the class but all the work must be completed by yourself.

Remember, DO NOT POST YOUR CODE PUBLICLY ON GITHUB! Any code found on GitHub that is not the base template you are given will be reported to SJA.
If you want to sidestep this problem entirely, don’t create a public fork and instead create a private repository to store your work.
