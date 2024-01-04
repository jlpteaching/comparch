---
Author: Jason Lowe-Power
Editor:  Maryam Babaie, Mahyar Samani
Title: ECS 201A Assignment 3
---

**Due on *{{ site.data.course.dates.gem5_3 }}* 11:59 pm (PST)**: See [Submission](#submission) for details

<img alt="Under construction" src="{{ "/img/under-construction.png" | relative_url }}">
Assignment coming soon

{% comment %}

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

Use [classroom: assignment 3](https://classroom.github.com/a/zntCWm3I) to create an assignment. You will be asked to **join**/**create** an assignment. If your teammate has already created an assignment, please **join** their assignment instead of creating one assignment. Otherwise, **create** your assignment and ask your teammate to **join** the assignment.

## Introduction

In this assignment, you'll be investigating the performance impacts of different out-of-order core designs on a set of RISC-V benchmarks.
The goals of this assignment are:

- Show how applications have different behaviors as the microarchitecture changes.
- Give you experience investigating the *bottleneck* in a particular architecture.
- Improve your understanding of out-of-order processor architecture.

## Workload

In this assignment, you are going to use 3 workloads for the evaluation of the performance of your systems.
Read more about each workload and how to use those workloads for your experiments below.

### Matrix Multiplication

You are going to use the same matrix multiplication program from [assignment 1]({{'modules/gem5/assignment1' | relative_url}}) as the first workload.
Please note that matrix size is hardcoded for this assignment.
You do not have to pass matrix size as an input argument to the `__init__` function for your workload.
Below you can find the C++ implementation of the matrix multiplication workload.

```cpp
#include <iostream>

#include "matrix.h"

#ifdef GEM5
#include "gem5/m5ops.h"
#endif


void multiply(double* A, double* B, double* C, int size)
{
    for (int i = 0; i < size; i++) {
        for (int k = 0; k < size; k++) {
            for (int j = 0; j < size; j++) {
                C[i * size + j] += A[i * size + k] * B[k * size + j];
            }
        }
    }
}

int main()
{
    std::cout << "Beginning matrix multiply ..." << std::endl;

#ifdef GEM5
    m5_work_begin(0,0);
#endif

    multiply(A, B, C, SIZE);

#ifdef GEM5
    m5_work_end(0,0);
#endif

    std::cout << "Finished matrix multiply." << std::endl;

    return 0;
}
```

Take a look at `workloads/matmul_workload.py` to learn more about instantiating a matrix multiplication workload.

### Breadth first search (**BFS**)

BFS is an algorithm for traversing a graph in breadth order.
The algorithm traverses vertices in a graph in an order corresponding to each vertex's relative depth to a specific vertex.
I.e. the algorithm starts traversing vertices in the graph starting from a specific vertex called `root`.
During the traversal, the algorithm makes sure to visit vertices that are only `1 edge/hop` away from `root` before it visits those vertices that are `2 edges/hops or more` away.
The BFS algorithm is one of the building blocks of graph analytics workloads.
Graph analytics workloads have common use cases such as modeling relationships and processes in physical, biological, social, and information systems.
[Beamer et al.](https://parlab.eecs.berkeley.edu/sites/all/parlab/files/main.pdf) discuss the BFS algorithm and how it could be parallized in their paper.
Below you can find a basic C++ implementation of the algorithm discussed in the paper.

```cpp
#include <iostream>
#include <vector>

#include "graph.h"

#ifdef GEM5
#include "gem5/m5ops.h"
#endif

int main()
{
    std::vector<int> frontier;
    std::vector<int> next;

    frontier.clear();
    next.clear();

    frontier.push_back(0);

    std::cout << "Beginning BFS ..." << std::endl;

#ifdef GEM5
    m5_work_begin(0,0);
#endif

    while (!frontier.empty()) {
        for (auto vertex: frontier) {
            int start = columns[vertex];
            int end = columns[vertex + 1];
            for (int i = start; i < end; i++) {
                int neighbor = edges[i];
                if (visited[neighbor] == 0) {
                    visited[neighbor] = 1;
                    next.push_back(neighbor);
                }
            }
        }
        frontier = next;
        next.clear();
    }

#ifdef GEM5
    m5_work_end(0,0);
#endif

    std::cout << "Finished BFS." << std::endl;

    return 0;
}
```

Take a look at `workloads/bfs_workload.py` to learn more about instantiating a BFS workload.

### Bubble Sort

Bubble sort is the starter algorithm for sorting arrays.
This program sorts an array by replacing each element with the smallest element to its right (if sorting ascendingly).
Below you can find the C++ implementation for bubble sort.

```cpp

#include <iostream>

#include "array.h"

#ifdef GEM5
#include "gem5/m5ops.h"
#endif

int main()
{
    std::cout << "Beginning bubble sort ... " << std::endl;

#ifdef GEM5
    m5_work_begin(0,0);
#endif

    for (int i = 0; i < ARRAY_SIZE - 1; i++) {
        for (int j = i + 1; j < ARRAY_SIZE; j++) {
            if (data[i] > data[j]) {
                int temp = data[i];
                data[i] = data[j];
                data[j] = temp;
            }
        }
    }

#ifdef GEM5
    m5_work_end(0,0);
#endif

    std::cout << "Finished bubble sort." << std::endl;

    return 0;
}

```

## Experimental setup

In this assignment, you are asked to design  your own out of order `processor` models for your experiments.
In regards to the rest of the components in your system:

- You will be using `HW3RISCVBoard` as your main `board` for your computer system.
You can find the model for the board in `components/boards.py`.
- You will be using `HW3MESICache` as your cache hierarchy for your computer system.
You can find its model in `components/cache_hierarchies.py`.
- You will be using `HW3DDR4` as your `memory` in your computer system.
You can find its model in `components/memories.py`.
- You will be using `2 GHz` as you clock frequency `clk_freq` in your system.

For your processor, you are going to use `HW3O3CPU` to model a high-performance and an efficient processor core.
`HW3O3CPU` is based on `O3CPU` which is an internal model of gem5.
Read up on the `O3CPU` in [gem5's documentation](https://www.gem5.org/documentation/general_docs/cpu_models/O3CPU).
Below, you can find details about `HWO3CPU` and its parameters.

### Pipeline width

For the purposes of this assignment, you need to configure a processor with the same width in all stages.
In the constructor of `HW3O3CPU` this attribute is named as `width`.

### Reorder Buffer size

This is the number of entries in the **r**e**o**rder **b**uffer (ROB).
This will constrain the number of instructions which can be "in flight" in the processor.
In the constructor of `HW3O3CPU` this attribute is named as `rob_size`.

### Number of physical registers

This is the number of registers in the *physical register file*.
A processor renames architecture registers to physical registers to resolve false dependences.
It also tracks true dependences (read-after-write) in the register file.
To learn more about register renaming, read up on [Tomasulo's algorithm](https://en.wikipedia.org/wiki/Tomasulo%27s_algorithm).

`HW3O3CPU` has two physical register files.
One register file for integer registers and one for floating point registers.
In the constructor of `HW3O3CPU`, `num_int_regs` refers to the number of *integer physical registers* and `num_fp_regs` refers to the number of *floating point physical registers*.

**NOTE**: Both register files must be larger than the 32 entries or gem5 will hang.
This is becuase the number of physical registers must be bigger than or equal to the number of logical registers.
RISC-V ISA defines 32 logical registers.

### big and LITTLE cores

In this assignment, you are required to design your own high-performance and efficient cores.
You need to add two core designs to `components/processors.py`.
In `componets/processors.py`, create a model based on `HW3O3CPU` and name it `HW3BigCore`.
This core will be your high-performance core for the assignment.
Create another model in `components/processors.py` and name it `HW3LittleCore`.
This core will be your efficient core for the assignment.
You can use information available on the internet on the different core microarchitectures to configure your `HW3BigCore` and `HW3LittleCore`.
As a starting point, take a look at [WikiChip](https://en.wikichip.org/wiki/WikiChip) and [AnandTech](https://www.anandtech.com/).
Your instructor has loosely modeled their `HW3BigCore` on [Intel Sunny Cove](https://en.wikichip.org/wiki/intel/microarchitectures/sunny_cove) and their `HW3LittleCore` on [Intel Gracemont](https://en.wikichip.org/wiki/intel/microarchitectures/gracemont).

**NOTE**: You are not required to match the specifications of any core.
Use information online as a guideline for your design.
You might find it impossible to match the specifications found online.
E.g. the base model `HW3O3CPU` assumes the same width for all the stages of the pipeline which is not usually the case with modern processor designs.

**Tips and To dos**: When designing your cores and caches, I recommend taking note of the following:

- When designing your cores, I strongly recommend **not** beefing up your cores, especially `HW3LittleCore`.
Remember that in computer design, there are almost always diminishing returns.
A beefy `HW3LittleCore` will result in a `HW3BigCore` that is not much more performant than `HW3LittleCore`.
I recommend dividing the values you see on specifications by in half for `HW3LittleCore`.
- Setting the `width` below 4 would result some unstability.
However, you might want to set `width` to 3 or less for `HW3LittleCore`.
I recommend starting with very little numbers for the rest of the parameters, especially `rob_size`, and gradually increasing them to get around this issue.
- Make sure you register files have more than 32 entries each.

## Analysis and simulation

Before running any simulations answer the following questions in your report.

1- What will be the average speed up of `HW3BigCore` over `HW3LittleCore`? Can you predict an upper bound using your pipeline parameters?
**Hint**: You can predict and upper bound for the speed up using Amdahl's law with optimistic values.

2- Do you think all the workloads will experience the same speed up between `HW3BigCore` and `HW3LittleCore`?

### Step I: Performance comparison

Now that you have completed the design process of `HW3BigCore` and `HW3LittleCore`, let's compare their performances using our 3 workloads as benchmarks.
Simulate each workload with each core.
For each workload compare the performance of `HW3BigCore` with the performance of `HW3LittleCore`.

Answer the following questions in your report.
Use relevant and correct reasoning in your answer.
Use simulation data with proper simulation data to strengthen your reasoning.

1. What is the speed up of `HW3BigCore` over `HW3LittleCore` for each workload?
2. What is the average improvement in IPC of `HW3BigCore` compared to `HW3LittleCore` over all the workloads?
**CAUTION**: Make sure to use the correct mean to report average IPC improvement.
3. Some workloads show more speedup that others. Which workloads show high speedup, which show low speedup? Look at the benchmark code (both the `.c` and `.s` files may be useful) and speculate the *algorithm characteristics* which influence the IPC difference between `HW3BigCore` and `HW3LittleCore`. What characteristics do applications have that lead to low performance improvement and what characteristics lead to high performance improvement?
4. Which workload has the highest IPC for `HW3BigCore`? What is unique about this workload?

**Hints**: Take a look at the assembly code of the **ROI** for inspiration.

### Step II: Medium core

In this step, you are tasked with finding a middle ground between `HW3BigCore` and `HW3LittleCore`.
This core needs to perform as closely as possible to `HW3BigCore` while using as little resources as `HW3LittleCore`.
We will refer to this core as `HW3MediumCore`.
To pick our sweet spot for the design of `HW3MediumCore`, we need to develop a methodology.
First, we need to define a cost function for increasing hardware resources.
We will use area as the cost of making a hardware.
Not all parameters of the pipeline have the same effect on the area.
E.g. Increasing the width of the pipeline has a quadratic effect on the area of the hardware while increasing register file entries has a linear effect.
Morever the cost of increasing two of these resources at the same time should be bigger than the sum of increasing each resource.
We will use an equation like below to score the area of a pipeline using the 4 parameters of `width`, `rob_size`, `num_int_regs`, and `num_fp_regs`.

$$ area_{score} = 2 * width^2 * rob\_ size + width^2 * (num\_ int\_ regs + num\_ fp\_ regs) + 4 * width + 2 * rob\_ size + (num\_ int\_ regs + num\_ fp\_ regs)$$

You can also get the area score for a pipeline design by calling the method `get_area_score` on the processor.

Now that we have our cost function, let's devise a method for measuring our gains.
In you report answer the following question.

1. If you were to use the speed up under only one workload from the 3 workloads you used before, which workload would you choose? Why?

Now that we have devised functions to measure costs and gains, configure 4 middle ground designs for the pipeline.
Not all of these designs will have the "best" cost-gain tradeoff.
In your report include the following figure.

2. Create a [pareto frontier](https://en.wikipedia.org/wiki/Pareto_efficiency) plot with cost on the y-axis and performance on the x-axis.
This will be a scatter plot with 6 points: the two "big" and "LITTLE" cores as well as your 4 middle ground designs.
Then, "connet the dots" on the "best" designs.

Answer this question in your report.

3. Assume you are an engineer working to design this middle ground core. Given this early analysis, which designs, if any, would you recommend your team to pursue developing? Explain why. (Note: you may want to annotate the above plot.)

## Submission

As mentioned before, you are allowed to submit your assignments in **pairs** and in **PDF** format.
You should submit your report on [gradescope](https://www.gradescope.com/courses/487868).
In your report answer the questions presented in [Analysis and simulation](#analysis-and-simulation), [Analysis and simulation: Step I](#step-i-performance-comparison), and [Analysis and simulation: Step II](#step-ii-medium-core).
Use clear reasoning and visualization to drive your conclusions.
Submit all your code through your assignment repository. Please make sure to include code/scripts for the following.

- `Instruction.md`: should include instruction on how to run your simulations.
- Automation: code/scripts to run your simulations.
- Configuration: python file configuring the systems you need to simulate.
You should add your final core designs to `components/processors.py`.
There should be 6 core definitions in `components/processor.py`.
They should include **1 design** for `HW3BigCore`, **1 design** for `HW3LittleCore`, and **4 designs** for `HW3MediumCore`.
You can add numbers to designs for `HW3MediumCore` to distinguish their design.
E.g. `HW3MediumCore0`, `HW3MediumCore1`, `HW3MediumCore2`, and `HW3MediumCore3`.

## Grading

Like your submission, your grade is split into two parts.

1. Reproducibility Package (50 points):
    a. Instruction and automation to run simulations for different section and dump statistics (20 points)
        - Instructions (5 points)
        - Automation (5 points)
    b. Configuration scripts and correct simulation setup (40 points): 10 points for configuration script(s) used to run your simulations and 5 points for implementing each of the 6 processor models as described in [Analysis and simulation: Step I](#step-i-performance-comparison), and [Analysis and simulation: Step II](#step-ii-medium-core).

2. Report (50 points): 5.5 points for each question presented in [Analysis and simualtion](#analysis-and-simulation), [Analysis and simulation: Step I](#step-i-performance-comparison), and [Analysis and simulation: Step II](#step-ii-medium-core).

## Academic misconduct reminder

You are required to work on this assignment in teams. You are only allowed to share you scripts and code with your teammate(s). You may discuss high level concepts with others in the class but all the work must be completed by your team and your team only.

Remember, DO NOT POST YOUR CODE PUBLICLY ON GITHUB! Any code found on GitHub that is not the base template you are given will be reported to SJA. If you want to sidestep this problem entirely, don’t create a public fork and instead create a private repository to store your work.

## Hints

- Start early and ask questions on Piazza and in discussion.
- If you need help, come to office hours for the TA, or post your questions on Piazza.

{% endcomment %}