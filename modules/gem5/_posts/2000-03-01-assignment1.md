---
Author: Jason Lowe-Power
Editor:  Maryam Babaie, Mahyar Samani
Title: ISA vs Technology
---

Originally from University of Wisconsin-Madison CS/ECE 752.

**Due on January 20 11:59 pm (PST)**: See [Submission](#submission) for details

GitHub Classroom link for 154B: [{{ site.data.course.154b_assignment1_invitation_link }}]({{ site.data.course.154b_assignment1_invitation_link }})
GitHub Classroom link for 201A: [{{ site.data.course.201a_assignment1_invitation_link }}]({{ site.data.course.201a_assignment1_invitation_link }})

## Table of Contents

- [Introduction](#introduction)
- [Workload](#workload)
- [Experimental setup](#experimental-setup)
- [Analysis and simulation](#analysis-and-simulation)
- [Submission](#submission)
- [Grading](#grading)
- [Academic misconduct reminder](#academic-misconduct-reminder)

## Introduction

In this assignment you are going to develop and run a set of experiments to test the effects of changing different components of a computer system on its performance.
We will use the Iron Law of Performance to guide our experiments.
We will investigate the effects of changing the ISA and the CPU frequency on the performance of a computer system.

Our research question is: **For a simple CPU model (i.e., fixed microarchitecture), does ISA or technology make a bigger impact on system performance?**

You are going to use a matrix multiplication program as the workload for your experiments.
Matrix multiplication is a commonly used kernel in many domains such as linear algebra, machine learning, and fluid dynamics.

## Workload

For this assignment we are going to use a matrix multiplication program as our workload.
The program takes and integer as input that determines the `size` of the square matrices `A`, `B`, and `C`.

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

We have provided three gem5 workloads for you that you can get with `obtain_resources`, one for RISC-V, one for Arm, and one for x86.
The names are:

- `matrix_multiply_riscv_run`
- `matrix_multiply_arm_run`
- `matrix_multiply_x86_run`

### Details of workloads

These workloads have gem5 region of interest (ROI) markers that allow you to measure the performance of the matrix multiplication program.
By default, at the beginning of the ROI the statistics will be reset.
At the end of the ROI, the statistics will be dumped to the `stats.txt` file.
Additionally, when gem5 exits, the statistics will be dumped to the `stats.txt` file.

> **CAVEAT [PLEASE READ CAREFULLY]**: When using these workloads with gem5, your simulation will output two sets of statistics in the same `stats.txt` file.
> Each set of statistics start with a line like below.

```text
---------- Begin Simulation Statistics ----------
```

Please make sure to **ignore** the **second** set of generated statistics in your analysis.

## Experimental setup

For this assignment, we will set up an experiment to see effect of changing a system's component on it performance.
You will need to write configuration scripts using gem5's stdlib that allow you to change the ISA, CPU and cache frequency.
Under the `components` directory, you will find modules that define the different models that you should use in your configuration scripts.

- Board models: You will using `RISCVBoard`, `X86Board`, and `ArmBoard` in this assignment.
- CPU models: You will be using `RISCVSingleCycleCPU`, `ArmSingleCycleCPU`, and `X86SingleCycleCPU` for this assignment.
- Cache models: You will be using `MESITwoLevelCache` in this assignment.
- Memory models: You will be using `DDR3` memory in this assignment.

Remember, you should use the `--outdir` option to specify the output directory for your simulation results so that they do not overwrite each other.

Each simulation should take between 1-5 minutes.

## Analysis and simulation

Complete the following steps and answer the questions for your report.
Collect data from your simulation runs and use simulator statistics to answer the questions.
Use clear reasoning and visualization to drive your conclusions.

Before starting with simulations, answer the following questions in your report.

### Step I: Write down your hypotheses and experimental setup

1. When you change the ISA from RISC-V to x86, what do you expect to happen to the performance of the system? Use the Iron Law of Performance to justify your answer.
2. When you change the CPU frequency from 1GHz to 4GHz, what do you expect to happen to the performance of the system? Use the Iron Law of Performance to justify your answer.
3. When you change the CPU frequency from 1GHz to 4GHz, will the speedup from 1GHz to 4GHz be the same for all ISAs? Why or why not?
4. Describe the experimental setup to answer the research question. What are the independent and dependent variables? What is the baseline?

### Step II: Investigating the impact of the ISA

Write a gem5 runscript that allows you to run either the RISC-V, Arm, or x86 matrix multiplication program.
Use a frequency of 1 GHz and DDR3 as the memory model.

In your report, answer the following questions after simulation supported with data.

1. What is the *performance* for matrix multiplication for each ISA?
2. Can you match the difference in performance to the Iron Law of Performance? Why or why not?

> **Hint**: In `workloads/matmul-basic` the makefile will generate the assembly code for the matrix multiplication program for each ISA based on the binary provided.
> You may find that useful to understand the differences between the ISAs.

### Step III: Investigating the impact of the CPU and cache clock frequency

Add the ability to change the frequency of the CPU and cache in your configuration script.
Use DDR3 as the memory model.

In your report, answer the following questions after simulation supported with data.

1. What is the speedup of the system when you change the CPU and cache frequency from 1GHz to 2GHz to 4GHz? Show the speedup for each ISA.
2. Does the Iron Law of Performance correctly predict the speedup for each ISA? Why or why not?

### Research question:

Answer the following question in your report based on the four steps above.

*For a simple CPU model (i.e., fixed microarchitecture), does ISA or technology make a bigger impact on system performance?*

We gathered data on the effects of the ISA with a fixed memory and frequency and the effects of the frequency with a fixed ISA and memory
Use the data you gathered to support your answer to the research question.

### Next steps (required 201A, extra credit 154B):

Answer the following questions in your report.

1. If the workload had a significantly better (lower) CPI, how would that change the results of the experiments? E.g., what would happen if the workload and microarchitecture supported a CPI of 0.25 (or 4 instructions per cycle)?

## Submission

You will submit this assignment via GitHub Classroom.

1. Accept the assignment by clicking on the link provided in the announcement.
2. Create a Codespace for the assignment on your repository.
3. Fill out the `questions.md` file.
4. Commit your changes.

Make sure you include both your runscript, an explanation of how to use your script, and the questions to the questions in the `questions.md` file.

### Explanation of how to use your script

Include a detailed explanation of how to use your script and how you use your script to generate your answers (this will be more applicable in future assignments).
Make sure that all paths are relative to this directory (`assignment-1/`).
The code included in the "Example command to run the script" section should be able to be copied and pasted into a terminal and run without modification.

- You should include a sentence or two which describes what the script (or scripts) do under "Explanation of the script" in `questions.md`.
- You should include the path to the script under "Script to run" in `questions.md`.
- You should include any parameters that need to be passed to the script under "Parameters to script (if any)" in `questions.md`.
- You should include each command used to gather data under "Command used to gather data" in `questions.md`.
  - Make sure this can by copy-pasted and run in your codespace without modification.
  - If you need other files to run your script, make sure to include those files when you commit your changes.

## Grading

- **25 points** gem5 runscript and explanation of how to use your script
- **50 points** for the questions in the report
- **25 points** for the research question
- **10 points** for the next steps

## Academic misconduct reminder

You are required to work on this assignment **individually**. You may discuss high level concepts with others in the class but all the work must be completed by you.

Remember, DO NOT POST YOUR CODE PUBLICLY ON GITHUB! Any code found on GitHub that is not the base template you are given will be reported to SJA. If you want to sidestep this problem entirely, don't create a public fork and instead create a private repository to store your work.