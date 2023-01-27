---
Author: Jason Lowe-Power
Editor:  Maryam Babaie, Mahyar Samani
Title: ECS 201A Assignment 3
---

**Due on *{{ site.data.course.dates.gem5_3 }}* 11:59 pm (PST)**: See [Submission](#submission) for details

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

Use [classroom: assignment 3]() to create an assignment. You will be asked to **join**/**create** an assignment. If your teammate has already created an assignment, please **join** their assignment instead of creating one assignment otherwise **create** your assignment and ask your teammate to **join** the assignment.

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

You are going to use the same matrix multiplication program in [assignment 1]({{'modules/gem5/assignment1' | relative_url}}) as the first workload.
Please note that matrix size is hardcoded for this assignment.
You do not have to pass matrix size as an input argument to the `__init__` function for your workload.

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
int main()
{
    std::vector<int> frontier;
    std::vector<int> next;

    frontier.clear();
    next.clear();

    frontier.push_back(0);

    while (!frontier.empty()) {
        for (auto vertex: frontier) {
            int start = columns[vertex];
            int end = columns[vertex + 1];
            for (int i = start; i < end; i++){
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

    return 0;
}
```

Take a look at `workloads/bfs_workload.py` to learn more about instantiating a BFS workload.

## Compile gem5 to execute RISC-V binaries

First, we need to create a gem5 simulator binary that can execute RISC-V code.
As of the writing of this assignment ({{ site.data.course.quarter }}), each *target* ISA that you want to run requires a different gem5 build.
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

## Template files and gem5's out-of-order CPU model

In this assignment, we will be using the [gem5 standard library](https://www.gem5.org/documentation/gem5-stdlib/overview) for the simulated system (called a board), to run the simulation via the `Simulator` class, and for the benchmark `Resources`.

Description of the files in the [template zip file]({{ "img/assignment3-template.tgz" | relative_url }}).

- `riscv_se.py`: This file contains a simple RISC-V system with a configurable out-of-order core.
- `stanford_benchmarks.py`: Contains the "resources" for the stanford benchmarks precompiled for RISC-V.
- `run.py`: This file is a simple gem5 run script which takes a parameter of the core (options found in the `cores` dictionary) and the benchmark (as specified in the `stanford_benchmarks.py` file.)
- `Stanford`: the stanford benchmarks. Each benchmark has an associated `.c` file which is the original source, a `.s` file which is the assembly generated by GCC, and the binary file that is executed in gem5. Each binary takes 1-3 minutes to execute in gem5.
- `run.sh`: A simple script to run all configurations for all benchmarks in parallel. By default, this will launch 22 processes, which is probably too many for most systems. Feel free to modify this to execute fewer at a time. This file also makes sure that the output directories for the stats files are all different based on the core name and the benchmark.

**NOTE**: in your `run.sh` file at line #26, you need to replace the `gem5-riscv` with the proper path of the `build/RISCV/gem5.opt` you compiled in the previous step.

### The O3CPU model

You can find some information on the O3CPU in gem5's documentation: <https://www.gem5.org/documentation/general_docs/cpu_models/O3CPU>.

### Options for the core design

#### Pipeline width

This is the width of the pipeline.
By setting this value, you're setting the width of each stage to be the same.

#### ROB size

This is the number of entries in the reorder buffer.
This will constrain the number of instructions which can be "in flight" in the processor.

#### Number of physical registers

This is the number of registers in the *physical* register file.
In order to overcome write-after-read and write-after-write hazards you have to be able to allocate temporary registers.
Register renaming and using a pool a physical register larger than the number of logical registers allows the out-of-order core to track the true read-after-write dependences and break some of the false dependences.

Note that this must be larger than the 32 logical registers in RISC-V or gem5 will hang.

#### Branch predictor

To find more information on these branch predictors, see `gem5/src/cpu/pred/BranchPredictor.py`.

- `simple`: Tournament branch predictor as discussed in class where there's a predictor to choose either a local or global predictor.
- `ltage`: A TAgged GEometric history length predictor. At a high-level, this branch predictor enables much longer (e.g., 100s or 1000s of instructions) global history. See <https://jilp.org/vol8/v8paper1.pdf> for details.
- `perceptron`: This branch predictor uses a table of simple neural networks (well, not a network because it's a single "neuron"). See <https://jilp.org/cbp2016/paper/DanielJimenez1.pdf> for more details.
- `complex`: Combines TAGE and Perceptron predictors.

### Given core configurations

You're given two core configurations.
You will have to extend this with other configurations as you are answering the questions below.

`LargeCore`: This is a core with many resources (some the maximum gem5 supports like being 8-wide). You will be using this as the "maximum" performance core.

`SmallCore`: This is a core with few resources (some the minimum that gem5 supports like only 48 physical registers). You will be using this as the "minimum" performance core.

## Assignment

Answer the following questions.
To answer the questions, you'll have to add other core configurations to the runscript, run experiments, and compare data between the experiments.
Be sure to include any relevant data in your writeup.
However, remember this should be *well-written* technical document, so *only* include relevant data and not extra data that's not needed.

### Part 1: Analyzing the workloads

1. What is the average improvement in IPC of the `LargeCore` compared to the `SmallCore`? (Remember to use the correct average statistic).
2. Some workloads show more speedup that others. Which workloads show high speedup, which show low speedup? Look at the benchmark code (both the `.c` and `.s` files may be useful) and speculate the *algorithm characteristics* which influence the IPC difference between the `LargeCore` and the `SmallCore`. What characteristics do applications have that lead to low performance improvement and what characteristics lead to high performance improvement?
3. Which workload(s) have the highest IPC for the large core? What is unique about this workload?

### Part 2: Analyzing the hardware

Between the `SmallCore` and the `LargeCore`, we are varying many parameters at the same time.
In this part, we will investigate which parameters are most important.
Vary the four parameters in the core configuration: pipeline width, ROB entries, physical register entries, and branch predictor to answer the following questions.

1. Which parameter(s) have the most impact on performance? Is this the same for all workloads or does it vary between different workloads?
2. Are there any dependences between the parameters (e.g., do you need to change two or more parameters at once to have an impact on the performance of different applications)?
3. Pick a core design between the cost of the `SmallCore` and the `LargeCore` which gives most of the performance benefits of the large core but with fewer resources.

    a. What are the parameters you chose? Let's call this the `GoodCore`.

    b. What is the average IPC improvement of your `GoodCore` design compared to the `SmallCore`?

    c. How much more performance could you get from the `LargeCore`? (What is the average speedup of the `LargeCore` compared to your `GoodCore`)?

    d. Are the workloads that benefited the most from the `LargeCore` the same as the ones that benefit from the `GoodCore`? Why or why not (talk about workload characteristics).


## Submission

For this assignment you don't need to turn in any code files. The only file you need to submit on gradescope at the designated section, is the pdf file of your report. Please do not forget to specify each question according to the outline when you're submitting your work.
In your report, you're not required to include any data which is not used in your analysis. Only include those that you actually use to justify your answer and make sure they are precisly and cleary specified.

## Grading

Grading breakdown is as follows:

Total points = 100

| #Question   | Points |
|-------------|--------|
| Part1.1	  | 10     |
| Part1.2	  | 15	   |
| Part1.3	  | 15     |
| Part2.1     | 10     |
| Part2.2	  | 10     |
| Part2.3.a	  | 10     |
| Part2.3.b	  | 10     |
| Part2.3.c	  | 10     |
| Part2.3.d	  | 10     |


## Academic misconduct reminder

You are to work on this project **individually**.
You may discuss *high level concepts* with one another (e.g., talking about the diagram), but all work must be completed on your own.

**Remember, DO NOT POST YOUR CODE PUBLICLY ON GITHUB!**
Any code found on GitHub that is not the base template you are given will be reported to SJA.
If you want to sidestep this problem entirely, don't create a public fork and instead create a private repository to store your work.
GitHub now allows everybody to create unlimited private repositories for up to three collaborators, and **you shouldn't have *any* collaborators** for your code in this class.

## Hints

* Start early! There is a learning curve for gem5, so start early and ask questions on Campuswire and in discussion.
* If you need help, come to office hours for the TA, or post your questions on Campuswire.
