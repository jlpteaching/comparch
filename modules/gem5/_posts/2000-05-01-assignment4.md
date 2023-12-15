---
Author: Jason Lowe-Power
Editor:  Maryam Babaie, Mahyar Samani
Title: ECS 201A Assignment 4
---

## Assignment 4 -- Due 11:59 pm (PST) *{{ site.data.course.dates.dino_4 }}*

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

Use [classroom: assignment 4](https://classroom.github.com/a/u2u6tuD8) to create an assignment. You will be asked to **join**/**create** an assignment. If your teammate has already created an assignment, please **join** their assignment instead of creating one assignment. Otherwise, **create** your assignment and ask your teammate to **join** the assignment.

## Introduction

In this assignment, you'll be investigating the performance impacts of different cache architectures and different algorithm designs on matrix multiplication.
The goals of this assignment are:

- Show how algorithms have different behaviors as the microarchitecture changes.
- Show how changing the algorithm can change performance on the *same* microarchitecture.
- Improve your understanding of cache architectures.

## Workload

In this assignment, you are going to implement different techniques for multiplying matrices.
All the ready made workloads discussed require one input argument.
You will have to pass `matrix_size` as an input argument to the constructor of a workload (`__init__`) which describes the size of matrices `A`, `B`, and `C`.
When running the workload on native hardware (e.g., your host machine), you should pass the same **one** argument in the command line following the name of your binary.

**NOTE**: for all the parts of this assignment assume:

$C = A \times B$

You will have to implement different techniques for matrix multiplications to see the effect of software implementation on the overall system performance.
Below you can see a short description of your starter code.

### Base matrix multiplication algorithm (C stationary, or ijk)

In programming, matrices are stored as two dimensional arrays.
Otherwise said, a matrix is stored as an array of arrays.
There are two ways how a matrix could be stored this way.

- Row major order: the matrix is represented as an array of the rows of the matrix.
Each row is then stored as an array that includes elements from consecutive columns in the row.
With this order, consecutive access to the elements in a row exhibits spatial locality.
- Column major order: the matrix is represented as an array of the columns of the matrix.
Each column is then stored as an array that includes elements from consecutive rows in the column.
With this order, consecutive access to the elements in a column exhibits spatial locality.

Below is a simple implementation of matrix multiplication given in `workloads/matmul/ijk_multiply.h`.
For this assignment, assume that all `A`, `B`, and `C` matrices are stored in row major order.
I.e., matrices are indexed like `A[row number][column number]`.
The starter code first iterates over the rows of `A` (*i*), then iterates over the columns of `B` (*j*), and then iterates over the elements in the selected row and column (*k*).

```cpp
void multiply(double **A, double **B, double **C, int size)
{
    for (int i = 0; i < size; i++) {
        for (int j = 0; j < size; j++) {
            for (int k = 0; k < size; k++) {
                C[i][j] += A[i][k] * B[k][j];
            }
        }
    }
}
```

The animation below shows how the order of operations during the multiplication operation.

![matrix multiplication]({{ '/img/mm_ijk.gif' | relative_url }})

You can import this workload into your configuration script from `workloads/matmul_workload.py` as `IJKMatMulWorkload`.

I have provided a ready made binary for this implementation.
However, if you need to compile your own binary, you can run the command below in `workloads/matmul`.

```sh
make mm-ijk-gem5
```

**CAUTION**: the above command generates a binary that could only be run with gem5.
Running this binaries on your host will result in errors.

### A stationary matrix multiplication (ikj)

As you can see in the animation, the previous implementation of the matrix multiplication does not expose much locality in its accesses to matrices `A` and `B`.
Rather, it exposes locality when accessing matrix `C`.
We can reorder our nested loops and still get the correct answer.
In fact all permutations of our three nested loops will result in the same `C` matrix.
Moreover, when it comes to the complexity of the algorithm itself, reordering the for loops does not change the complexity.
However, reordering the for loops, changes the access pattern to the memory.
In turn, it could increase/decrease our cache hit rate.
For this step, you can see a slightly different implementation of the matrix multiplication program below.
You can also find it in `workloads/matmul/ikj_multiply.h`

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

As you can see, two for loops have been interchanged (*j* and *k*) from our previous implementation.
The animation below shows how the order of operations during the multiplication operation.

![matrix multiplication]({{ '/img/mm_ikj.gif' | relative_url }})

You can import this workload into your configuration script from `workloads/matmul_workload.py` as `IKJMatMulWorkload`.

I have provided a ready made binary for this implementation.
However, if you need to compile your own binary, you can run the command below in `workloads/matmul`.

```shell
make mm-ikj-gem5
```

**CAUTION**: the above command generates a binary that could only be run with gem5.
Running this binaries on your host will result in multiple errors.

### Blocked matrix multiplication

You can improve the cache behavior of matrix multiplication by using a blocked algorithm.
In this algorithm, rather than streaming through all of the inputs, you operate on one *block* at a time.
Look at [this short article](https://csapp.cs.cmu.edu/public/waside/waside-blocking.pdf) to learn more about the blocking technique used to increase locality in matrix multiplication.

Similar to loop interchange, there are multiple different ways you can choose to block accesses to your matrices in the matrix multiplication algorithm.
One example is shown below where `k` and `j` are blocked and `i` is streamed.

![blocked matrix multiplication]({{ '/img/mm_blocked.gif' | relative_url }})

For this assignment, you need to implement three different variants of the blocking scheme.

- Block *i* and *j* and implement it as the multiply function in `workloads/matmul/block_ij_multiply.h`.
After implementation, you can build your binary by running the command below in `workload/matmul`.

```shell
make mm-block-ij-gem5
```

After building your binary, you can import it as a workload into your configuration script from `workloads/matmul_workload.py` as `BlockIJMatMulWorkload`.

- Block *i* and *k* and implement it as the multiply function in `workloads/matmul/block_ik_multiply.h`.
After implementation, you can build your binary by running the command below in `workload/matmul`.

```shell
make mm-block-ik-gem5
```

After building your binary, you can import it as a workload into your configuration script from `workloads/matmul_workload.py` as `BlockIKMatMulWorkload`.

- Block *k* and *j* and implement it as the multiply function in `workloads/matmul/block_kj_multiply.h`.
After implementation, you can build your binary by running the command below in `workload/matmul`.

```shell
make mm-block-kj-gem5
```

After building your binary, you can import it as a workload into your configuration script from `workloads/matmul_workload.py` as `BlockKJMatMulWorkload`.

You will have to pass `matrix_size` (describes the size of matrices `A`, `B`, and `C`) and `block_size` (describes the size of the block in your blocking scheme) to the constructor of all of the workloads you have implemented.
If you have to run a workload on native hardware (e.g. your host machine), you should pass the same **two** arguments in the same order in the command line following the name of your binary.

**NOTE**: You can use the command below in `workloads/matmul` to create the binaries for all the workloads discussed.

```shell
make all-gem5
```

**CAUTION**: the above command generates binaries that could only be run with gem5.
Running these binaries on your host will result in multiple errors.

### Testing your blocked implementation

To test your blocked implementation you may want to run it on your local machine (or on the codespace you're using).
To do this, see [Running on native hardware](#step-iv-running-on-native-hardware) below.

## Experimental setup

In this assignment, we will investigate the effect of caching on overall performance.
We have already looked at how software implementation can help improve caching in matrix multiplication.
In regards to hardware models, we will use different cache hierarchies to see the effect of cache size and latency on the performance.
Under the `components` directory, you will find modules that define the different models that you should use in your configuration scripts.

- Board models: You will find the definitions for `HW4RISCVBoard` in `components/boards.py`.
- Board models: You will find the definitions for `HW4O3CPU` in `components/processors.py`.
- Cache models: You can find all the models you need to use for your cache hierarchy under `components/cache_hierarchies.py`.
You can find three models for your cache hierarchy.
They all have an L2 cache with the size of `128 KiB`.
They also have the same L1I cache.
However, there have different L1D cache design.
You can find a short description of each L1D design below.
  - HW4LargeCache: a `48 KiB` L1D cache with higher latency.
  - HW4MediumCache: a `32 KiB` L1D cache with medium latency.
  - HW4SmallCache: a `16 KiB` L1 cache with lower latency.

Make sure you understand their similarities and differences.

- Memory models: You will find the definitions for `HW4DDR4` in `components/memories.py`.
- Clock frequency: You can use a clock frequency of `2 GHz` for all of your simulations.

### **IMPORTANT NOTE**

In your configuration scripts, make sure to import `exit_event_handler` using the command below.

```python
from workloads.roi_manager import exit_event_handler
```

You will have to pass `exit_event_handler` as a keyword argument named `on_exit_event` when creating a `simulator` object. Use the *template* below to create a simulator object.

```python
simulator = Simulator(board={name of your board}, full_system=False, on_exit_event=exit_event_handler)
```

## Analysis and simulation

As part of your analysis and simulation you will have to run your workloads on both real hardware and gem5.
Before running any workloads, let's take a look at our working set size.
Working set size is the number of bytes that have to be moved between the processor and the memory subsystem.

### Step I: Working set sizes

In your report answer the following questions.

1. What is the working set size for the matrix multiply application?
Describe the working set size as a function of `matrix_size` and size of a double `double_size`.
Plug in 128 as `matrix_size` and 8 as `double_size` and calculate the working set size.
2. For each of the three blocking configurations, what's the *active working set* for multiplying *one block*?
Describe the working set size as a function of `matrix_size`, `block_size`, and `double_size`.
Plug in 128 as `matrix_size`, 8 as `block_size`, and 8 as `double_size`.

### Step II: Simulation and performance comparison

For your simulation, create a configuration script that allows you to run any of the workloads with any of the cache hierarchies.
For this step:

- run all the workloads on `HW4SmallCacheHierarchy`
- run `BlockIJMatMulWorkload` with all the cache hierarchies.

In your report answer the following questions.

1. For `HW4SmallCacheHierarchy`, which blocking scheme performs the best? Why?
2. For `BlockIJMatMulWorkload`, which cache hierarchy performs the best? Why?

Use a `matrix_size` of 128 and a `block_size` of 8 for all your simulations.

**Hints**: In your answer use cache related statistics such as `hit rate`.
Moreover, think of how **a**verage **m**emory **a**ccess **t**ime (**AMAT**) could help you explain your results.
I also recommend thinking about how the access pattern could influence memory access time.

For this step, you will run **6 configurations** in total.

### Step III: Finding an optimal setup

In this step, you will use your conclusions in the last step to predict a combination of blocking scheme and cache hierarchy that performs best.
In your report, answer the following questions.

1. What combination of a blocking scheme and cache hierarchy will perform best?
In your answer describe your approach for finding this combination.
Remember, don't exhaustively search, but try to use the information and the statistics from the previous step to find the best combination.
2. Between the size and the latency of the caches which one did you find has a more significant effect?

### Step IV: Running on native hardware

Now, run the matrix multiply workloads on some real hardware (not gem5).
You can run it on the CSIF machines, your laptop, or the codespaces computer.
To build all your binaries for your native hardware (e.g., your host), use the command below in `workloads/matmul`.

```shell
make all-native
```

You can also build them separately using the commands below.

```shell
make mm-block-ij-native
```

```shell
make mm-block-ik-native
```

```shell
make mm-block-kj-native
```

Before running your worklods on real hardware, answer the following questions in your report.

1. What is the L1/L2/L3 size of the processor you're running on? (`lscpu` or `/proc/cpuinfo` and Google should help)
2. Can you use the information about the cache sizes to predict the best performing block scheme and size?

When running on native hardware, I recommend using `matrix_size` of at least as big as 256.
After running the workloads on real hardware answer the following question in your report.

3. Which blocking scheme and size exhibited the best performance? Is this the same as on gem5? Why or why not?

## Submission

As mentioned before, you are allowed to submit your assignments in **pairs** and in **PDF** format.
You should submit your report on [gradescope](https://www.gradescope.com/courses/487868).
In your report answer the questions presented in , [Analysis and simulation: Step I](#step-i-working-set-sizes), [Analysis and simulation: Step II](#step-ii-simulation-and-performance-comparison), [Analysis and simulation: Step III](#step-iii-finding-an-optimal-setup), and [Analysis and simulation: Step IV](#step-iv-running-on-native-hardware).

Use clear reasoning and visualization to drive your conclusions.
Submit all your code through your assignment repository. Please make sure to include code/scripts for the following.

- `Instruction.md`: should include instruction on how to run your simulations.
- Automation: code/scripts to run your simulations.
- Configuration: python file configuring the systems you need to simulate.
- Workload implementation: You will need to add your implementations for the three different blocking schemes to their respective source code files in `workloads/matmul`.

## Grading

Like your submission, your grade is split into two parts.

1. Reproducibility Package (50 points):
    a. Instruction and automation to run simulations for different section and dump statistics (20 points)
        - Instructions (5 points)
        - Automation (5 points)
    b. Configuration scripts and source codes (40 points): 5 points for configuration script(s) used to run your simulations as described in [Analysis and simulation: Step II](#step-ii-simulation-and-performance-comparison), 10 points for implementing each of the 3 blocking schemes as described in [Blocked matrix multiplication](#blocked-matrix-multiplication), and 5 points for a scripts used to run workloads on native hardware as described in [Analysis and simulation: Step IV](#step-iv-running-on-native-hardware).

2. Report (50 points): 5.5 points for each question presented in [Analysis and simulation: Step I](#step-i-working-set-sizes), [Analysis and simulation: Step II](#step-ii-simulation-and-performance-comparison), [Analysis and simulation: Step III](#step-iii-finding-an-optimal-setup), and [Analysis and simulation: Step IV](#step-iv-running-on-native-hardware).

## Academic misconduct reminder

You are required to work on this assignment in teams. You are only allowed to share you scripts and code with your teammate(s). You may discuss high level concepts with others in the class but all the work must be completed by your team and your team only.

Remember, DO NOT POST YOUR CODE PUBLICLY ON GITHUB! Any code found on GitHub that is not the base template you are given will be reported to SJA. If you want to sidestep this problem entirely, don’t create a public fork and instead create a private repository to store your work.

## Hints

- Start early and ask questions on Piazza and in discussion.
- If you need help, come to office hours for the TA, or post your questions on Piazza.

{% endcomment %}