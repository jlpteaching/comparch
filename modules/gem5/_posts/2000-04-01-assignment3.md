---
Author: Jason Lowe-Power
Editor:  Maryam Babaie, Mahyar Samani, Kaustav Goswami
Title: Out-of-Order Core Designs
---

**Due on *{{ site.data.course.dates.gem5_3 }}* 11:59 pm (PST)**: See [Submission](#submission) for details.

GitHub Classroom link for 154B: [{{ site.data.course.154b_assignment3_invitation_link }}]({{ site.data.course.154b_assignment3_invitation_link }})

GitHub Classroom link for 201A: [{{ site.data.course.201a_assignment3_invitation_link }}]({{ site.data.course.201a_assignment3_invitation_link }})

## Table of Contents

- [Introduction](#introduction)
- [Research Question](#research-question)
- [Workloads](#workloads)
- [Experimental setup](#experimental-setup)
- [The Base model](#the-base-model)
- [Your experiments](#your-experiments)
- [Submission](#submission)
- [Grading](#grading)
- [Academic misconduct reminder](#academic-misconduct-reminder)

## Introduction

In this assignment, you'll be investigating the performance impacts of different out-of-order core designs on a set of RISC-V benchmarks.
The goals of this assignment are:

- Show how applications have different behaviors as the microarchitecture changes.
- Give you experience investigating the *bottleneck* in a particular architecture.
- Improve your understanding of out-of-order processor architecture.

## Research Question

**What is the bottleneck in the performance of an out-of-order processor?**

You will be given a design of an out-of-order processor.
Your task is to investigate the performance of this processor on a set of benchmarks and determine what the main bottleneck is.
Then, you will design a new processor that improves the performance of the original processor with the minimal impact on area.

The questions you will be answering in this assignment are:

1. What is the biggest bottleneck in the performance of the original processor? The width, the number of physical registers, or the reorder buffer size?
2. What is the design of the processor that gives the most performance improvement with the least impact on area?

## Workloads

In this assignment, you are going to use four workloads for the evaluation of the performance of your systems.
Read more about each workload and how to use those workloads for your experiments below.

You will be using the following workloads for your experiments detailed below.

- Matrix Multiplication (`matrix_multiply_riscv_run`)
- Breadth First Search (`bfs_riscv_run`)
- Bubble Sort (`bubble_riscv_run`)
- N-Queens (`queens_riscv_run`)

Note that all of these workloads have been compiled with the gem5 annotations enabled.

Together, these workloads are called the "comparch-benchmarks".
This is a *suite* of workloads to use for your evaluation.
When using `obtain_resource` you can also get a *suite* which is a list of workloads.
You can iterate over the suite to run all the workloads in the suite using the *multisim* feature of gem5.

These workloads run in approximately 1-10 minutes each.

> Note: Multisim in gem5 24.1 [has a couple of bugs](https://github.com/gem5/gem5/issues/1738).
> However, it should work if you use 1 or 2 simultaneous simulations.

### Matrix Multiplication

You are going to use the same matrix multiplication program from [assignment 1](../assignment-1/assignment.md) as the first workload.
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

### Breadth first search (**BFS**)

BFS is an algorithm for traversing a graph in breadth order.
The algorithm traverses vertices in a graph in an order corresponding to each vertex's relative depth to a specific vertex.
I.e. the algorithm starts traversing vertices in the graph starting from a specific vertex called `root`.
During the traversal, the algorithm makes sure to visit vertices that are only `1 edge/hop` away from `root` before it visits those vertices that are `2 edges/hops or more` away.
The BFS algorithm is one of the building blocks of graph analytics workloads.
Graph analytics workloads have common use cases such as modeling relationships and processes in physical, biological, social, and information systems.
[Beamer et al.](https://parlab.eecs.berkeley.edu/sites/all/parlab/files/main.pdf) discuss the BFS algorithm and how it could be parallelized in their paper.
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

### N-Queens

The N-Queens problem is a classic computer science problem.
It involves placing N chess queens on an N×N chessboard so that no two queens threaten each other.
This means that no two queens can share the same row, column, or diagonal.
The most common method to solve the N-Queens problem is to use backtracking.
Below you can find the C++ implementation for the same:

```cpp
#include <bits/stdc++.h>

#ifdef GEM5
#include "gem5/m5ops.h"
#endif

void printSolution(int **board, int N) {
    for (int i = 0; i < N; i++) {
        for (int j = 0; j < N; j++)
            printf(" %d ", board[i][j]);
        printf("\n");
    }
}

bool isSafe(int **board, int row, int col, int N) {
    for (int i = 0; i < col; i++)
        if (board[row][i])
            return false;

    for (int i=row, j=col; i>=0 && j>=0; i--, j--)
        if (board[i][j])
            return false;

    for (int i=row, j=col; j>=0 && i<N; i++, j--)
        if (board[i][j])
            return false;

    return true;
}

bool solveNQUtil(int **board, int col, int N) {
    if (col >= N)
        return true;

    for (int i = 0; i < N; i++) {
        if (isSafe(board, i, col, N)) {
            board[i][col] = 1;

            if (solveNQUtil(board, col + 1, N))
                return true;

            board[i][col] = 0;
        }
    }

    return false;
}

bool solveNQ(int **board, int N) {


    if (solveNQUtil(board, 0, N) == false) {
        printf("Solution does not exist");
        return false;
    }

    return true;
}

int main(int argc, char *argv[]) {
    // the user needs to input the size of the chessboard
    if (argc == 2) {
        int size = atoi(argv[1]);

        // initialize the board
        int **board = new int*[size];
        for (int i = 0 ; i < size; i++)
            board[i] = new int[size];

        for (int i = 0 ; i < size ; i++) {
            for (int j = 0 ; j < size ; j++) {
                board[i][j] = 0;
            }
        }
#ifdef GEM5
    m5_work_begin(0,0);
#endif
        solveNQ(board, size);
#ifdef GEM5
    m5_work_end(0,0);
#endif
        printSolution(board, size);
    }
    else {
        printf("N-Queens program. Usage \n $ ./queens <chess-board-size>\n");
    }
    return 0;
}
```

We use an input of 18 for the chessboard size in the workload.

## Experimental setup

In this assignment, you will use the following base configurations for your computer system.

- You will be using `RISCVBoard` as your main `board` for your computer system.
You can find the model for the board in `components/boards.py`.
- You will be using `MESITwoLevelCache` as your cache hierarchy for your computer system.
You can find its model in `components/cache_hierarchies.py`.
- You will be using `DDR4` as your `memory` in your computer system.
You can find its model in `components/memories.py`.
- You will be using `1 GHz` as you clock frequency `clk_freq` in your system.

For your processor, you are going to use `OutOfOrderCPU` to model a high-performance and an efficient processor core.
`OutOfOrderCPU` is based on `O3CPU` which is an internal model of gem5.
Read up on the `O3CPU` in [gem5's documentation](https://www.gem5.org/documentation/general_docs/cpu_models/O3CPU).
Below, you can find details about `OutOfOrderCPU` and its parameters.

### Pipeline width

For the purposes of this assignment, you need to configure a processor with the same width in all stages.
In the constructor of `OutOfOrderCPU` this attribute is named as `width`.
The width is one of the main constraints for how many instructions can be processed in parallel.

### Reorder Buffer size

This is the number of entries in the **r**e**o**rder **b**uffer (ROB).
This will constrain the number of instructions which can be "in flight" in the processor.
In the constructor of `OutOfOrderCPU` this attribute is named as `rob_size`.
The number of in flight instructions is also a main contraint for how many instructions can be processed in parallel.
In this case, if instructions take more than 1 cycle to execute, you need to have enough space in the ROB to keep track of the instructions.
If you don't have enough space in the ROB, the processor will stall.

### Number of physical registers

This is the number of registers in the *physical register file*.
A processor renames architecture registers to physical registers to resolve false dependences.
It also tracks true dependences (read-after-write) in the register file.
To learn more about register renaming, read up on [Tomasulo's algorithm](https://en.wikipedia.org/wiki/Tomasulo%27s_algorithm).

`OutOfOrderCPU` has two physical register files.
One register file for integer registers and one for floating point registers.
In the constructor of `OutOfOrderCPU`, `num_int_regs` refers to the number of *integer physical registers* and `num_fp_regs` refers to the number of *floating point physical registers*.

**NOTE**: Both register files must be larger than the 32 entries or gem5 will hang.
This is because the number of physical registers must be bigger than or equal to the number of logical registers.
RISC-V ISA defines 32 logical registers.

The number of physical registers can also constrain the amount of parallelism in the processor.
If there are not enough physical registers, the processor will stall.

So, you need to balance the width, the number of physical registers, and the ROB size to get the best performance per unit area or per unit power.
The processor is *out of balance*, then you will be wasting resources and not getting the best performance.

## The base model

You are already given a run script which runs the "comparch-benchmarks".
We have also included the output for two different configurations of the `OutOfOrderCPU` in the `results` directory.

The first configuration we call the "little" core, and your job is to find the main bottlenecks in this design.
The second configuration we call the "big" core.
The "big" core has a large width, a large ROB size, and a large number of physical registers, leading to a very large area.
You can see below that the speedup of the "big" core over the "little" core is not as large as you might expect.
The details of the configurations and the performance results are shown below.

| Core Type | Width | ROB Size | Number of Integer Registers | Number of Floating Point Registers |
|-----------|-------|----------|-----------------------------|------------------------------------|
| little    | 4     | 32       | 64                          | 64                                 |
| big       | 8     | 192      | 256                         | 256                                |

> Note: There's a bug in gem5's out-of-order model such that when running with small widths sometimes there's an assertion failure.
> Therefore, we have set the width to 4 for the "little" core.

| Workload           | Little Core IPC | Big Core IPC | Speedup |
|--------------------|------------------|--------------|---------|
| BFS                | 0.525410         | 0.809980     |   1.54      |
| Bubble Sort        | 0.879611         | 0.895811     |   1.02      |
| Matrix Multiplication | 1.608251      | 1.685272     |   1.05      |
| N-Queens           | 1.000015         | 1.122047     |   1.12      |

### Running the base model

Now that you have to run multiple workloads for multiple configurations, you need to automate the process.
You can use the `multisim` feature of gem5 to run multiple simulations in parallel.

> Note: `multisim` has some bugs, so if you experience hangs, you can run workloads one-by-one.
> However, we still suggest using `multisim` even if you manually run experiments because it helps keep things organized.

We have provided a script `experiments.py` that runs the big and little designs.
You can extend this script to run your experiments.

In `experiments.py`, we first define a function to get the board.
Then, we have a set of configurations that we want to run.
You can uncomment the next lines to print the area of each configuration (used to answer some of the questions).
Finally, there is a nested for loop that will run each benchmark for each configuration.
In this for loop, instead of calling `simulator.run()` we add the simulator to the multisim object which will run all the simulations in parallel when we run the script with the multisim module (as shown below).

```python
from components import (
    RISCVBoard,
    MESITwoLevelCache,
    DDR4,
    OutOfOrderCPU,
)

from gem5.simulate.simulator import Simulator
from gem5.utils.multisim import multisim
from gem5.resources.resource import obtain_resource

multisim.set_num_processes(10)

def get_board(width, rob_size, num_int_regs, num_fp_regs):
    cache = MESITwoLevelCache()
    memory = DDR4()
    cpu = OutOfOrderCPU(width, rob_size, num_int_regs, num_fp_regs)

    board = RISCVBoard(
        clk_freq="1GHz", processor=cpu, cache_hierarchy=cache, memory=memory
    )

    return board

configurations = {
    "little": {"width": 4, "rob_size": 32, "num_int_regs": 64, "num_fp_regs": 64},
    "big": {"width": 12, "rob_size": 384, "num_int_regs": 512, "num_fp_regs": 512},
}

# for name, config in configurations.items():
#     board = get_board(**config)
#     print(f"Area of {name} configuration: {board.get_processor().get_area_score()}")
# exit(0)

for workload in obtain_resource("comparch-benchmarks"):
    for name in ["little", "big"]:
        config = configurations[name]
        board = get_board(**config)
        board.set_workload(workload)
        simulator = Simulator(board=board, id=f"{name}-{workload.get_id()}")
        multisim.add_simulator(simulator)
```

To run the script with the multisim module, you can use the following command.

```bash
gem5 -re -m gem5.utils.multisim experiments.py
```

When you run this, you will find the output in the `m5out` directory with one directory for each simulation.
The names come from the `id` parameter of the `Simulator` object.
We also use the `-re` flag to redirect the stdout and stderr of each simulation to a file in the `m5out` directory.

You can list all of the names of the simulations with the following command.

```bash
gem5 experiments.py --list
```

Then, you can run a specific simulation with the following command.

```bash
gem5 experiments.py <id>
```

It is particularly useful to run a single simulation if you make changes to the parameters and need to re run just a small subset of the simulations.

## Your experiments

Remember, the research question you are trying to answer is:
What is the bottleneck in the performance of an out-of-order processor?

### Step 1: Choose your configurations

You run *three* experiments to answer this question to determine if the bottleneck is the width, the number of physical registers, or the reorder buffer size.
You will need to run the following experiments:

- A wide processor with a small reorder buffer and a small number of physical registers.
- A processor with a small width, a large reorder buffer, and a small number of physical registers.
- A wide processor with a large reorder buffer and a small number of physical registers.

1. What are the configurations you are going to run?

### Step 2: Run your experiments

Run your experiments using the `experiments.py` script.
After running the experiments, compile the results in a table.
Include the speedup of each configuration over the "little" core.

1. What is the speedup of each configuration over the "little" core? Present your table as described above.

### Step 3: Analyze your results and answer the research question

1. What is the biggest bottleneck in the performance of the original processor? The width, the number of physical registers, or the reorder buffer size? Use your data to support your answer.
2. Using the `board.get_processor().get_area_score()` method, calculate the area of each configuration. What is the area-efficient design?

### Step 4: Workload analysis

The workloads show different behaviors with some more sensitive to the width of the processor and others more sensitive to the number of physical registers or the ROB size.

1. Describe which workload is affected more by each parameter of your configuration.
2. [Required 201A, extra credit 154B] Use the application's source code, the resulting assembly code, or other information to explain why each workload is more sensitive to a particular parameter.

## Submission

You will submit this assignment via GitHub Classroom.

1. Accept the assignment by clicking on the link provided in the announcement.
2. Create a Codespace for the assignment on your repository.
3. Fill out the `questions.md` file.
4. Commit your changes.

Make sure you include both your runscript, an explanation of how to use your script, and the questions to the questions in the `questions.md` file.

### Explanation of how to use your script

Include a detailed explanation of how to use your script and how you use your script to generate your answers.
The code included in the "Example command to run the script" section should be able to be copied and pasted into a terminal and run without modification.

- You should include a sentence or two which describes what the script (or scripts) do under "Explanation of the script" in `questions.md`.
- You should include the path to the script under "Script to run" in `questions.md`.
- You should include any parameters that need to be passed to the script under "Parameters to script (if any)" in `questions.md`.
- You should include each command used to gather data under "Command used to gather data" in `questions.md`.
  - Make sure this can by copy-pasted and run in your codespace without modification.
  - If you need other files to run your script, make sure to include those files when you commit your changes.

## Grading

- **25 points** gem5 experiment script and explanation of how to use your script
- **75 points** for the questions in the report
- **25 points** for the next steps

## Academic misconduct reminder

You are required to work on this assignment individually.
You may discuss high level concepts with others in the class but all the work must be completed by yourself.

Remember, DO NOT POST YOUR CODE PUBLICLY ON GITHUB! Any code found on GitHub that is not the base template you are given will be reported to SJA. If you want to sidestep this problem entirely, don’t create a public fork and instead create a private repository to store your work.
