---
Author: Mahyar Samani, Kaustav Goswami, Jason Lowe-Power
Title: Getting started with gem5
---

**Due on *{{ site.data.course.dates.gem5_0 }}* 11:59 pm (PST)**: See [Submission](#submission) for details.

## Table of Contents

- [Introduction](#introduction)
- [gem5 and gem5's standard libary](#gem5-and-gem5s-standard-library-gem5-stdlib)
- [Writing your own configuration script](#writing-your-own-configuration-script)
- [Workload and gem5-resources](#workload-and-gem5-resources)
- [Final step: simulate](#final-step-simulate)
- [Invoking gem5](#invoking-gem5)
- [Submission](#submission)
- [Grading](#grading)
- [Academic misconduct reminder](#academic-misconduct-reminder)

## Introduction

In this assignment, we will use gem5's standard library to simulate the `Hello World` program in C++ on gem5.
You will learn how to write your own configuration scripts to describe the computer system you need to simulate and pass your workloads and benchmark to the simulator.

## gem5 and gem5's standard library (gem5-stdlib)

gem5 is a well known event based simulator for computer architecture research.
You will learn about simulation and its different techniques in class and during the discussions.
The gem5 simulator is designed so that is very powerful in describing widely varying systems, but this means it is also quite complex.
With the purpose of simplifying system description, gem5's standard library has been developed to allow users to use ready made components and user friendly interfaces to describe their system.
The gem5 standard library (stdlib) is the digital world equivalent of going to your local BestBuy and picking the different components you need to build your computer system.
Although, the components along with their names do not match the commercial products of Intel, AMD, and Nvidia.

### `gem5.components.boards`

Each system in gem5 is described as a `board`.
You can think of this `board` as the **motherboard** in a computer system.
In this assignment we are going to be using a ready made board in gem5's standard library that we have renamed to `RISCVBoard`.
This board will use a RISC-V-based processor.
You can find the definition of `RISCVBoard` in `components/__init__.py`.
You can see that it is based on `SimpleBoard` from the standard library.
You can find the `SimpleBoard` source code and documentation in [`gem5/src/python/gem5/components/boards/simple_board.py`](/gem5/src/python/gem5/components/boards/simple_board.py).

Similar to a real system, a system in gem5 is not complete with only a `board`.
However, the `board` is the platform through which all the other components communicate with each other.
In order to build a complete system we need 3 other components: a "processor", a "cache hierarchy", and a "memory".
This is not exactly the same way you would build a real computer system.
For real computers, "cache hierarchy" comes pre-packaged with the "processor" and you need other components such as a **power supply** and a **storage drive** to complete a real system.
However, there are many research focuses on cache hierarchy design that has motivated this separation of "processor" and "cache hierarchy".

In addition, you need to specify the clock frequency for the processor and cache hierarchy to the board.

### `gem5.components.processors`

Ready made `Processor` objects in the standard library represent the processing cores in a real CPU.
In this assignment, we are going to use the `SingleCycleCPU`.
You can find its source code in this repo in `components/__init__.py`.
This processor is based on `SimpleProcessor` from the standard library.
Whenever instantiated, it will create a `SimpleProcessor` with **one** `TimingSimpleCPU` core with `RISC-V` instruction set architecture (ISA).
The `TimingSimpleCPU` is a simple in-order CPU model that is designed to be fast to simulate.
It assumes *no* time to execute instructions and only models the time to fetch the instruction from the cache.
In practice, this results in a approximately 1 cycle for each ALU operation.
For memory operations, the time to get the data from memory is modeled based on the cache hierarchy and memory system.
In other words, `TimingSimpleCPU` models a single-cycle processor except for memory operations, which can be more than a cycle.

You can find the source code to `SimpleProcessor` in [`gem5/src/python/gem5/components/processors/simple_processor.py`](/gem5/src/python/gem5/components/processors/simple_processor.py).
In addition, it is highly recommended that you learn more about gem5's different CPU models in the links below.

- [SimpleCPU](https://www.gem5.org/documentation/general_docs/cpu_models/SimpleCPU)
- [MinorCPU](https://www.gem5.org/documentation/general_docs/cpu_models/minor_cpu)
- [O3CPU](https://www.gem5.org/documentation/general_docs/cpu_models/O3CPU)

### `gem5.components.cachehierarchies`

The `cache hierarchy` objects in standard library are designed to implement **cache coherency protocol** for multi-core processors and model the **interconnect** between multiple cores and multiple memory channels.
In this assignment we are going to use `MESITwoLevelCache`.
You can find its source code in `components/__init__.py`.

This cache hierarchy is based on `MESITwoLevelCacheHierarchy` from the standard library.
You can find the source code to `MESITwoLevelCacheHierarchy` in [`gem5/src/python/gem5/components/cachehierarchies/ruby/mesi_two_level_cache_hierarchy.py`](/gem5/src/python/gem5/components/cachehierarchies/ruby/mesi_three_level_cache_hierarchy.py).
Whenever instantiated, a `MESITwoLevelCache` creates a **two level** cache hierarchy with the **MESI** protocol for **cache coherency**.
It has a **64KiB 8-way set associative L1 instruction cache**, **64KiB 8-way set associative L1 data cache**, **256KiB 4-way set associative L2 unified cache with 16 banks**.

### `gem5.components.memory`

The `memory` objects in standard library are designed to implement various types of single and multi-channel DRAM based memories.
In this assignment we are going to use `DDR3`.
You can find its source code in `components/memories.py`.
This memory is based on `ChanneledMemory` from the standard library.
You can find the source code to `ChanneledMemory` in [`gem5/src/python/gem5/components/memory/memory.py`](/gem5/src/python/gem5/components/memory/memory.py).
Whenever instantiated, a `DDR3` creates a **single channel DDR3 DRAM memory with 1GiB of capacity and a data bus frequency of 1600MHz**.

## Writing your own configuration script

We will write configuration scripts that describe our desired system to be simulated in python.
We will go through the following steps to complete our configuration script.
Before that, let's create a script called **run.py** in the assignment's base directory.
We will keep adding to this script until it is complete.
Then we will pass this script to the gem5 binary for simulation.

### Import models

We will need to import the models for the different components that we aim to simulate.
Here is the code that imports all the mentioned models for ``.

```python
from components import RISCVBoard, SingleCycleCPU, MESITwoLevelCache, DDR3
```

### Instantiate an object of the model

In this step we will create an object of each model. We will first create a `processor` and name it **cpu**, then we will create a `cache hierachy` and name it **cache**, then we will create a `memory` and name it **memory**, and then we will create a `board` and name it **board**.
We will need **cpu, cache, and memory** to create a board. Here is the code that does that for us.

```python
if __name__ == "__m5_main__":
    cpu = TimingSimpleCPU()
    cache = MESITwoLevelCache()
    memory = DDR3()
    board = RISCVBoard(
        clk_freq="2GHz", processor=cpu, cache_hierarchy=cache, memory=memory
    )
```

## Workload and gem5-resources

So far, we have written a configuration script that describes the system we would like to simulate.
However, our simulation setup is not complete.
We still need to describe what **software** needs to be executed on the **hardware** that we just created.
In computer architecture research frequently used programs and kernels (small pieces of code essential to many programs e.g. quicksort) are used to evaluate the performance of a computer system.
These programs usually come in a package and are referred to as **benchmarks**. There are many benchmarks available.
[SPEC2017](https://spec.org/cpu2017/) and [PARSEC](https://parsec.cs.princeton.edu/) are among the popular **benchmarks** in computer architecture research.

[gem5 resources](https://resources.gem5.org/) is a project aiming to offer ready made resources compatible with the gem5 simulator.
You can download and use compiled binaries from many benchmarks and small programs from gem5-resources.

In this assignment, as mentioned before, we are going to use an already compiled `Hello World` binary for the RISC-V ISA from gem5-resources.
There are many workloads available in gem5-resources, but we have provided some for you in the [`resources.json` file](../workloads/resources.json).

### Using a workload in gem5

In order to use a workload in gem5, you need to download the workload from gem5-resources.
You can do that by using the `obtain_resource` function from the `gem5.resources` package.

```python
from gem5.resources.resource import obtain_resource
...
workload = obtain_resource("hello_world_workload")
```

### Setting the workload for simulation

Next, we will need to create an object of the workload we just imported and describe that this workload object is the object that we want to use for the **software** on our specified **hardware**.
You can do that by calling `set_workload` function from `RISCVBoard`.
Here is the line of code that does that.

```python
board.set_workload(workload)
```

## Final step: simulate

The last step before our configuration script is complete is to create a simulator object.
The simulator object allows us to give directions to gem5 on specific things we need gem5 to do for simulation.
You will see examples of interacting with the simulator later on.
We can create a simulator object through an internal python package from gem5.
The simulate package allows user to easily set up the simulation environment for the experiments.
**NOTE**: Most of gem5's internal python packages work exclusively with gem5.
gem5 has an internal modified python interpreter that these packages use to communicate with gem5.
The following steps show how to import the simulator package and instantiate a simulator object.

### Import simulator

In order to import the simulator package, add the following line to you configuration script.

```python
from gem5.simulate.simulator import Simulator
```

### Create a simulator object

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
simulator = Simulator(board=board)
```

The last action item is to tell gem5 to run the simulation.
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
from components import RISCVBoard, SingleCycleCPU, MESITwoLevelCache, DDR3

from gem5.resources.resource import obtain_resource
from gem5.simulate.simulator import Simulator

if __name__ == "__m5_main__":
    cpu = SingleCycleCPU()
    cache = MESITwoLevelCache()
    memory = DDR3()
    board = RISCVBoard(
        clk_freq="2GHz", processor=cpu, cache_hierarchy=cache, memory=memory
    )
    board.set_workload(obtain_resource("hello_world_workload"))

    simulator = Simulator(board=board)
    simulator.run()
    print(f"Simulation finished.")
```

## Invoking gem5

As discussed earlier, gem5 has a modified internal python interpreter.
Please note that passing your configuration scripts to python will result in numerous confusing errors.
You will need to pass your configuration script to the gem5 binary to run your simulation.
To do that you will need to use the command line.
Now, open a terminal and run the command below.

```shell
gem5 run.py
```

After running this command in the terminal, you will see an output like below:

```text
Global frequency set at 1000000000000 ticks per second
src/mem/dram_interface.cc:690: warn: DRAM device capacity (8192 Mbytes) does not match the address range assigned (1024 Mbytes)
src/base/statistics.hh:279: warn: One of the stats is a legacy stat. Legacy stat is a stat that does not belong to any statistics::Group. Legacy stat is deprecated.
src/arch/riscv/isa.cc:279: info: RVV enabled, VLEN = 256 bits, ELEN = 64 bits
src/base/statistics.hh:279: warn: One of the stats is a legacy stat. Legacy stat is a stat that does not belong to any statistics::Group. Legacy stat is deprecated.
src/base/remote_gdb.cc:418: warn: Sockets disabled, not accepting gdb connections
gem5 Simulator System.  https://www.gem5.org
gem5 is copyrighted software; use the --copyright option for details.

gem5 version 24.1.0.0
gem5 compiled Dec  9 2024 21:50:01
gem5 started Dec 18 2024 00:14:19
gem5 executing on c5955d84feb8, pid 1
command line: gem5 run.py

info: Standard input is not a terminal, disabling listeners.
info: Using config file specified by $GEM5_CONFIG
info: Using config file at /workspaces/gem5-assignment-template/gem5-config.json
src/sim/simulate.cc:199: info: Entering event queue @ 0.  Starting simulation...
src/mem/ruby/system/Sequencer.cc:704: warn: Replacement policy updates recently became the responsibility of SLICC state machines. Make sure to setMRU() near callbacks in .sm files!
src/sim/syscall_emul.hh:1117: warn: readlink() called on '/proc/self/exe' may yield unexpected results in various settings.
      Returning '/workspaces/gem5-assignment-template/assignment-0/resources/riscv-hello'
src/sim/mem_state.cc:448: info: Increasing stack size by one page.
Hello world!
Simulation finished.
```

You can see that our print statement has produced the output that we expected.
If you see an output like the one above, congratulations.
You just completed your first simulation with gem5.

After running gem5, you will find there is a new directory created, named `m5out`.
This directory includes all the simulator output files.
You can find the statistics output under `m5out/stats.txt`.
It includes many statistics about your simulated hardware.
The file is human readable.
Please take the time to take a look at the first 15 lines of that file and understand what each statistic means.
**NOTE**: `host` statistics refer to statistics about the actual machine that the simulation was run on and `guest`/`simulated` statistics refer to the statistics of the simulated computer system.

### Redirecting simulator output

Like every other program gem5 uses standard out `stdout` and standard error `stderr` to communicate some of it information with the user.
However, you can tell gem5 to not dump `stdout` and `stderr` to your terminal and output those to a file.
To do this add `-re` flag to your previous command you used to run your simulation.
Make sure to add this flag before the name of your configuration script as the flag should be passed to gem5 and not your configuration script.
Below is how the command looks like after adding `-re` flag to it.

```shell
gem5 -re run.py
```

Now, if you look at `m5out`, you will see that there is a new file name `simout.txt` and `simerr.txt`.
Let's print the content of that file and compare to our previous output in [Invoking gem5](#invoking-gem5).
To do that, run the following command in your terminal.

```shell
cat m5out/simout.txt
```

Below you can see the output after running the command above.

```text
Global frequency set at 1000000000000 ticks per second
Redirecting stdout to m5out/simout.txt
Redirecting stderr to m5out/simerr.txt
gem5 Simulator System.  https://www.gem5.org
gem5 is copyrighted software; use the --copyright option for details.

gem5 version 24.1.0.0
gem5 compiled Dec  9 2024 21:50:01
gem5 started Dec 18 2024 00:18:14
gem5 executing on 1ef68e3397df, pid 1
command line: gem5 -re run.py

info: Standard input is not a terminal, disabling listeners.
info: Using config file specified by $GEM5_CONFIG
info: Using config file at /workspaces/gem5-assignment-template/gem5-config.json
Hello world!
Simulation finished.
```

You can see that apart from the date and time of simulation the two outputs look similar.

You can also print the content of `m5out/simerr.txt` to see the rest of the output that gem5 has produced.

```text
src/mem/dram_interface.cc:690: warn: DRAM device capacity (8192 Mbytes) does not match the address range assigned (1024 Mbytes)
src/base/statistics.hh:279: warn: One of the stats is a legacy stat. Legacy stat is a stat that does not belong to any statistics::Group. Legacy stat is deprecated.
src/arch/riscv/isa.cc:279: info: RVV enabled, VLEN = 256 bits, ELEN = 64 bits
src/base/statistics.hh:279: warn: One of the stats is a legacy stat. Legacy stat is a stat that does not belong to any statistics::Group. Legacy stat is deprecated.
src/base/remote_gdb.cc:418: warn: Sockets disabled, not accepting gdb connections
src/sim/simulate.cc:199: info: Entering event queue @ 0.  Starting simulation...
src/mem/ruby/system/Sequencer.cc:704: warn: Replacement policy updates recently became the responsibility of SLICC state machines. Make sure to setMRU() near callbacks in .sm files!
src/sim/syscall_emul.hh:1117: warn: readlink() called on '/proc/self/exe' may yield unexpected results in various settings.
      Returning '/workspaces/gem5-assignment-template/assignment-0/resources/riscv-hello'
src/sim/mem_state.cc:448: info: Increasing stack size by one page.
```

In this file you can see the rest of gem5's output.
It includes warning and whatever gem5 has outputted to `stderr`.

Note that there are a number of warnings.
You can safely ignore the DRAM device capacity, legacy stats, sockets disabled, and replacement policy updates warnings.
You should be aware of the readlink warning which means that the simulation you are running will *not* be deterministic like most gem5 simulations since you are actively accessing host files.
All of the `info` blocks are just for your information and can be safely ignored.

### Changing the output directory

Since gem5 uses the same name `m5out` for its output directory every time you run gem5, the simulator output will be overwritten every time you run gem5.
However, gem5 allows you to manually change this directory to a directory of your liking.
To do that, you have to add `--outdir=[path to your desired directory]` to your command that you use to invoke gem5.
Make sure to pass it to gem5 and not your configuration script.
This means you have to put `--outdir=[path to your desired directory]` before the path to your configuration script.
This is what an example command with `--outdir` will look like.

```shell
gem5 --outdir=test run.py
```

After running the command above, you will see that a directory with the name `test` has been created.

If you look at the content of `test`, you will see that it includes all the simulator outputs that you could find in `m5out` before.

### Passing input arguments to configuration script

gem5 allows you to pass input arguments to your configuration script.
This way you will not need to create a configuration script for every system you want to simulate.
You can pass your input arguments to your configuraion script following the path to your configuration script.
gem5 will then capture that part of the command line and pass it to your configuration script.
You will then have to parse that part of the command line to read your input arguments from the command line.
Below is how you can invoke gem5 with the flags we discussed before and input arguments to your configuration script.

```shell
gem5 [-re] {path to your configuration script} [input arguments to your configuration script]
```

I highly recommend you to read up on [argparse](https://docs.python.org/3/library/argparse.html).
It is a feature rich python package that allows you to parse the command line.

## Submission

You will submit this assignment via GitHub Classroom.

1. Accept the assignment by clicking on the link provided in the announcement.
2. Create a Codespace for the assignment on your repository.
3. Fill out the `questions.md` file.
4. Commit your changes.

Make sure you include both your runscript, an explanation of how to use your script, and the questions to the questions in the `questions.md` file.

### Explanation of how to use your script

Include a detailed explanation of how to use your script and how you use your script to generate your answers (this will be more applicable in future assignments).
Make sure that all paths are relative to this directory (`assignment-0/`).
The code included in the "Example command to run the script" section should be able to be copied and pasted into a terminal and run without modification.

- You should include a sentence or two which describes what the script (or scripts) do under "Explanation of the script" in `questions.md`.
- You should include the path to the script under "Script to run" in `questions.md`.
- You should include any parameters that need to be passed to the script under "Parameters to script (if any)" in `questions.md`.
- You should include the command used to gather data under "Command used to gather data" in `questions.md`.
  - Make sure this can by copy-pasted and run in your codespace without modification.
  - If you need other files to run your script, make sure to include those files when you commit your changes.

## Grading

- **50 points** `run.py` script located at `assignment-0/run.py`
- **25 points** for the `questions.md` file located at `assignment-0/questions.md`
- **25 points** for explanation of how to use your script located at `assignment-0/questions.md`

## Academic misconduct reminder

You are required to work on this assignment **individually**. You may discuss high level concepts with others in the class but all the work must be completed by you.

Remember, DO NOT POST YOUR CODE PUBLICLY ON GITHUB! Any code found on GitHub that is not the base template you are given will be reported to SJA. If you want to sidestep this problem entirely, don't create a public fork and instead create a private repository to store your work.
