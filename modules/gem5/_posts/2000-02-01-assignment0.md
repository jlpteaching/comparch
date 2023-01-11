---
Author: Mahyar Samani
Title: ECS 201A Assignment 0
---

# Assignment 0 -- Due 11:59 pm (PST) *{{ site.data.course.dates.gem5_0 }}* (0 Points)

**Due on *{{ site.data.course.dates.gem5_0 }}* 11:59 pm (PST)**: See [Submission](#submission) for details.

## Table of Contents

- [Introduction](#introduction)
- [gem5 and gem5's standard libary](#workload)
- [Experimental setup](#experimental-setup)
- [Analyzing simulation data](#analyzing-simulation-data)
- [Submission](#submission)
- [Grading](#grading)
- [Academic misconduct reminder](#academic-misconduct-reminder)
- [Hints](#hints)

## Introduction

You should complete this assignment **individually**. This assignment is not graded and will not influence your grade. However, it is highly encouraged to complete this assingment as soon as you can and get started on the next assignments. Make sure to start early and post any questions you might have on Piazza.

In this assignment, we will use gem5's standard library to simulate the `Hello World` program in c++ on gem5. You will learn how to write your own configuration scripts to describe the computer system you need to simulate and pass your workloads and benchmark to the simulator.

## gem5 and gem5's standard library (gem5-stdlib)

gem5 is a well known event based simulator for computer architecture research.
You will learn about simulation and its different techniques in class and during the discussions.
The gem5 simulator is designed so that is very powerful in describing widely varying systems, but this means it is also quite complex.
With the purpose of simplifying system description, gem5's standard library has been developed to allow users to use ready made components and user friendly interfaces to describe their system.
The gem5 standard libary (stdlib) is the digital world equivalent of going to your local BestBuy and picking the different components you need to build your computer system.
Although, the names of these components do not match the commercial products of Intel, AMD, and Nvidia.

### `gem5.components.boards`

Each system in gem5 is described as a `board`.
You can think of this `board` as the **motherboard** in a computer system. 
In this assignment we are going to be using a ready made board in gem5's standard library that we have renamed to `HW0RISCVBoard`.
This board will use a RISC-V-based
You can find the definition of `HW0RISCVBoard` in this repo in `components/boards.py`.
You can see that it is based on `SimpleBoard` from the standard library.
You can find the `SimpleBoard` source code and documentation in `gem5/src/python/gem5/components/boards/simple_board.py`.

Similar to a real system, a system in gem5 is not complete with only a `board`.
However, the `board` is the platform through which all the other components communicate with each other.
In order to build a complete system we need 3 other components: a `processor`, a `cache hierarchy`, and a `memory`.
This is not exactly the same way you would build a real computer system.
For real computers, `cache hierarchy` comes pre-packaged with the `processor` and you need other components such as a **power supply** and a **storage drive** to complete a real system. 
However, there are many research focuses on cache hierarchy design that has motivated this separation of `processor` and `cache hierarchy`.

In addition, you need to specify the clock frequency for the `processor` and `cache hierarchy` to the `board`.

### `gem5.components.processors`

Ready made `processor` objects in the standard library represent the processing cores in a real CPU.
In this assignment, we are going to use `HW0TimingSimpleCPU`.
You can find its source code in this repo in `components/processors.py`.
This processor is based on `SimpleProcessor` from the standard libary.
Whenever instantiated, it will create a `SimpleProcessor` with **one** `TimingSimpleCPU` core with `RISC-V` instruction set architecture (ISA).
You can find the source code to `SimpleProcessor` in `gem5/src/python/gem5/components/processors/simple_processor.py`.
In addition, it is highly recommended that you learn more about gem5's different CPU models in the link below.

- [SimpleCPU](https://www.gem5.org/documentation/general_docs/cpu_models/SimpleCPU)
- [MinorCPU](https://www.gem5.org/documentation/general_docs/cpu_models/minor_cpu)
- [O3CPU](https://www.gem5.org/documentation/general_docs/cpu_models/O3CPU)

### `gem5.components.cachehierarchies`

The `cache hierarchy` objects in standard libary are designed to implement **cache coherency protocol** for multi-core processors and model the **interconnect** between multiple cores and multiple memory channels.
In this assignment we are going to use `HW0MESITwoLevelCache`.
You can find its source code in `components/cache_hierarchies.py`.
This cache hierarchy is based on `MESITwoLevelCacheHierarchy` from the standard library.
You can find the source code to `MESITwoLevelCacheHierarchy` in `gem5/src/python/gem5/components/cachehierarchies/ruby/mesi_two_level_cache_hierarchy.py`.
Whenever instantiated, a `HW0MESITwoLevelCache` creates a **two level** cache hierarchy with the **MESI** protocol for **cache coherency**.
It has a **64KiB 8-way set associative L1 instruction cache**, **64KiB 8-way set associative L1 data cache**, **256KiB 4-way set associative L2 unified cache with 16 banks**.

### `gem5.components.memory`

The `memory` objects in standard library are designed to implement various types of single and multi-channel DRAM based memories.
In this assignment we are going to use `HW0DDR3_1600_8x8`.
You can find its source code in `components/memories.py`.
This memory is based on `ChanneledMemory` from the standard library.
You can find the source code to `ChanneledMemory` in `gem5/src/python/gem5/components/memory/memory.py`.
Whenever instantiated, a `HW0DDR3_1600_8x8` creates a **single channel DDR3 DRAM memory with 1GiB of capacity and a data bus frequency of 1600MHz**.

## Writing your own configuration script

We will write configuration scripts that describe our desired system to be simulated in python.
We will go through the following steps to complete our configuration script.
Before that, let's create a script called **run.py** in the assignment's base directory.
We will keep adding to this script until it is complete.
Then we will pass this script to the gem5 binary for simulation.

### Import models

We will need to import the models for the different components that we aim to simulate.
Here is the code that imports all the mentioned models for `HW0`.

```python
from components.boards import HW0RISCVBoard
from components.processors import HW0TimingSimpleCPU
from components.cache_hierarchies import HW0MESITwoLevelCache
from components.memories import HW0DDR3_1600_8x8
```

### Instantiate an object of the model

In this step we will create an object of each model. We will first create a `processor` and name it **cpu**, then we will create a `cache hierachy` and name it **cache**, then we will create a `memory` and name it **memory**, and then we will create a `board` and name it **board**.
We will need **cpu, cache, and memory** to create a board. Here is the code that does that for us.

```python
if __name__ == "__m5_main__":
    cpu = HW0TimingSimpleCPU()
    cache = HW0MESITwoLevelCache()
    memory = HW0DDR3_1600_8x8()
    board = HW0RISCVBoard(
        clk_freq="2GHz", processor=cpu, cache_hierarchy=cache, memory=memory
    )
```

## Workload and gem5-resources

So far, we have written a configuration script that describes the system we would like to simulate.
However, our simulation setup is not complete.
We still need to make calls to the simulator and start the simulation.
However, even after adding those calls, our simulation setup is not complete.
We still need to describe what **software** needs to be executed on the **hardware** that we just described.
In computer architecture research frequently used programs and kernels (small pieces of code essential to many programs e.g. quick sort) are used to evaluate the performance of a computer system.
These programs usually come in a package and are referred to as **benchmarks**. There are many benchmarks available.
[SPEC2017](https://spec.org/cpu2017/) and [PARSEC](https://parsec.cs.princeton.edu/) are among the popular **benchmarks** in computer architecture research.

[gem5 resources](https://resources.gem5.org/) is a project aiming to offer ready made resources compatible with the gem5 simulator.
You can download and use compiled binaries from many benchmarks and small programs from gem5-reosources.

In this assignment, as mentioned before, we are going to use an already compiled `Hello World` binary for the RISC-V ISA from gem5-resources.
You can find the source code to `HelloWorldWorkload` in `workloads/hello_world.py`.

### Importing workload

We need to import our `HelloWorldWorkload` to our configuration script.
To do that add the following line to your import section of the code.

```python
from workloads.hello_world_workload import HelloWorldWorkload
```

### Setting the workload for simulation

Next, we will need to create an object of the workload we just imported and describe that this workload object is the object that we want to use for the **software** on our specified **hardware**.
You can do that by calling `set_workload` function from `HW0RISCVBoard`.
Here is the line of code that does that.

```python
workload = HelloWorldWorkload()
board.set_workload(workload)
```

Here is how the configuration script looks like so far.

```python
from components.boards import HW0RISCVBoard
from components.processors import HW0TimingSimpleCPU
from components.cache_hierarchies import HW0MESITwoLevelCache
from components.memories import HW0DDR3_1600_8x8

from workloads.hello_world_workload import HelloWorldWorkload

if __name__ == "__m5_main__":

    cpu = HW0TimingSimpleCPU()
    cache = HW0MESITwoLevelCache()
    memory = HW0DDR3_1600_8x8()
    board = HW0RISCVBoard(
        clk_freq="2GHz", processor=cpu, cache_hierarchy=cache, memory=memory
    )
    workload = HelloWorldWorkload()
    board.set_workload(workload)
```

## Final step: simulate

The last step before our configuration script is complete is to create a simulator object.
We will do that through an internal python package from gem5.
The simulate package allows user to easily set up the simulation environment for the experiments.
**NOTE**: Most of gem5's internal python packages work exculsively with gem5.
gem5 has an internal modified interpreter that these packages use to communicate with gem5.
The following steps show how to import the simulator package and instantiate a simulator object.

### Import simulator

In order to import the simulator package, add the following line to you configuration script.

```python
from gem5.simulate.simulator import Simulator
```

### Create a simulator object and run the simulation

Next, we will create a simulator object.
In order to create a simulator object we need to specify what the system to be simulated is and what mode of simulation should be used.
We have already described the system to be simulated in the previous steps.
We will need to only pass **board** as the representative for the whole system.
In regards to simulation modes, gem5 works in two simulation modes. 
The two modes are referred to as Full System (FS) mode and Syscall Emulation (SE) mode.
FS mode is like "bare metal" simulation which requires a kernel and disk image to *boot* Linux and extra script to then run some application.
In SE mode, gem5 "fakes" the system calls and they take 0 time.
However, SE mode doesn't require a disk or kernel image, it only requires a binary (usually statically compiled).
So SE mode is a little easier to get going than FS mode.

We will be using SE mode for this assignment.
This means that we need to pass value `False` as the `full_system` argument to our simulator. 
Here is the piece of code that creates a simulator object.

```python
simulator = Simulator(board=board, full_system=False)
```

### Run the simulation

The last action item is to run the simulation.
Make a call to `run` function from the simulator to do that.
Here is the piece of code that calls this function.

```python
simulator.run()
```

Let's add a print statement to show the simulation is over.
Add the following line to your configuration script.

```python
print("Finished simulation.")
```

Here is how the script looks like after adding all the necessary statements.

```python
from components.boards import HW0RISCVBoard
from components.processors import HW0TimingSimpleCPU
from components.cache_hierarchies import HW0MESITwoLevelCache
from components.memory import HW0DDR3_1600_8x8

from workloads.hello_world_workload import HelloWorldWorkload

if __name__ == "__m5_main__":

    cpu = HW0TimingSimpleCPU()
    cache = HW0MESITwoLevelCache()
    memory = HW0DDR3_1600_8x8()
    board = HW0RISCVBoard(
        clk_freq="2GHz", processor=cpu, cache_hierarchy=cache, memory=memory
    )
    workload = HelloWorldWorkload()
    board.set_workload(workload)
    simulator = Simulator(board=board, full_system=False)
    simulator.run()

    print("Finished simulation")
```
