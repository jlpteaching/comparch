---
Author: Jason Lowe-Power
Editor:  Maryam Babaie
Title: ECS 201A Assignment 2
---

# Assignment 2 -- Due 11:59 pm (PST) Sunday, January 23rd 2022 (48 Points)

Originally from University of Wisconsin-Madison CS/ECE 752 .

Modified for ECS 201A, Winter 2022.

**Due on 01/23/2022 11:59 pm (PST)**: See [Submission](#submission) for details

## Table of Contents
  - [Introduction](#introduction)
  - [Step I](#step-i)
  - [Step II](#step-ii)
  - [Step III](#step-iii)
  - [Step IV](#step-iv)
  - [Submission](#submission)
  - [Grading](#grading)
  - [Academic misconduct reminder](#academic-misconduct-reminder)
  - [Hints](#hints)

## Introduction

You should do this assignment on your own, although you are welcome to talk to classmates in person or on Campuswire about any issues you may have encountered. The standard late assignment policy applies -- you may submit up to 1 day late with a 10% penalty.

The purpose of this assignment is to give you experience with pipelined CPUs. You will simulate a given program with TimingSimple CPU to understand the instruction mix of the program. Then, you will simulate the same program with a pipelined in-order CPU to understand how the latency and bandwidth of different parts of pipeline affect performance. You will also be exposed to pseudo-instructions that are used for carrying out functions required by the underlying experiment. This homework is based on exercise 3.6 of CA:AQA 3rd edition (the former textbook for this course) and was developed in part by Jason Lowe-Power et al., then modernized by Matt Sinclair and Jason Lowe-Power.

## Step I
The DAXPY loop (double precision aX + Y) is an oft used operation in programs that work with matrices and vectors. The following code implements DAXPY in C++14 (please compile your program for c++14).

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

Your first task is to compile this code statically and simulate it with gem5 using the *TimingSimple CPU*. You can name the config file you use to run it with gem5 as `run.py`.

In your report, report the breakup of instructions for different op classes -- and provide a brief analysis of the breakdown. For this, grep for statExecutedInstType in the stats.txt file (Note: there are also summary stats directly above the statExecutedInstType stats that you may find useful). You should also use the same two-level cache configuration as assignment1.

## Step II

Generate the assembly code for the DAXPY program above by using the -S and -O3 options when compiling with g++. As you can see from the assembly code, instructions that are not central to the actual task of the program (computing aX + Y) will also be simulated. This includes the instructions for generating the vectors X and Y, summing elements in Y, and printing the sum. When we compiled the code with -S, we got about 320 lines of assembly code with O2 and 500 lines of assembly code with O3, with only about 15-20 lines for the actual DAXPY loop. 

Optional: you may find -fverbose-asm useful to include in your Makefile for this step.

Usually while carrying out experiments for evaluating a design, one would like to look only at statistics for the portion of the code that is most important. This part of the code is also known as the region of interest. To look only at the region of interest, typically programs are annotated so that the simulator, on reaching the beginning of an annotated portion of the code, will carry out functions like create a checkpoint, output, and reset statistical variables. By doing this, it ensures that our stats are representative for the region of the code we care about, instead of mixing these stats in with the stats for parts we aren't focused on (e.g., generating the vectors in DAXPY).

To learn how to reset the stats in gem5, you will edit the C++ code from the [Step I](#step-i) to output and reset stats just before the start of the DAXPY loop and just after it. For this, include the file [m5op.h](https://gem5.googlesource.com/public/gem5/+/refs/heads/stable/include/gem5/m5ops.h) in the program (you will find this file in include/gem5/ directory of the gem5 repository). Use the function m5_dump_reset_stats() from this file in your program. This function outputs the statistical variables and then resets them. You can provide 0 as the value for both the delay and the period arguments. If you want to learn more about m5ops, [here](https://www.gem5.org/documentation/general_docs/m5ops/) is a good place to start. In particular, you will need to update your CFLAGS and LDFLAGS similarly to how the documentation does.

Execute `scons build/x86/out/m5` in the $GEM5_ROOT/util/m5/ directory. This will create a file named libm5.a (and a binary named m5) in $GEM5_ROOT/util/m5/build/x86/out. Link this library with the program for DAXPY (compile with g++, see above for CFLAGS and LDFLAGS updates). Now again simulate the program with the TimingSimple CPU. This time you should see three sets of statistics in the stats.txt file.

**NOTE**: If you're using CSIF machines, do not forget to use this format: **$HOME/.local/bin/scons** for scons.

In your report, report the breakup of instructions among different opcode classes for the three parts of the program. Provide the fragment of the generated assembly code that starts with call to m5_dump_reset_stats() and ends with another call to m5_dump_reset_stats(), with the main DAXPY loop in between.

## Step III

As the tutorial with assignment1 discussed, there are several different types of CPUs that gem5 supports: atomic, TimingSimple, out-of-order, in-order and KVM. Let's talk about the timing and in-order CPUs. The TimingSimple CPU executes each arithmetic instruction in a single cycle, but requires multiple cycles for memory accesses. Also, it is not pipelined. So only a single instruction is being worked upon at any time. The in-order CPU (also known as MinorCPU) executes instructions in a pipelined fashion with the following pipe stages: fetch1, fetch2, decode and execute. Remember, as discussed in assignment1, you must add MinorCPU to the command line to get it to compile.

Especially if you didn't already for assignment1, take a look at the file MinorCPU.py. In the definition of MinorFU, the class for functional units, we define two quantities opLat and issueLat. From the comments provided in the file, understand how these two parameters are to be used. Also note the different functional units that are instantiated as defined in class MinorDefaultFUPool.

Assume that the issueLat and the opLat of the FloatSimdFU can vary from 1 to 6 cycles and that they always sum to 7 cycles (e.g., issueLat = 1, opLat = 6, 1+6=7). For each decrease in the opLat, we need to pay with a unit increase in issueLat (e.g., issueLat = 2, opLat = 5). 

In your report, answer: which design of the FloatSimd functional unit would you prefer? Provide statistical evidence obtained through simulations of the annotated portion of the code.

You can find a skeleton file that extends the minor CPU [here](http://pages.cs.wisc.edu/~sinclair/courses/cs752/fall2021/handouts/hw/hw2/cpu.py). If you use this file, you will have to modify your config scripts (run.py) to work with it. Also, you'll have to modify this file to support the next part.

## Step IV

The Minor CPU has by default two integer functional units as defined in the file MinorCPU.py (ignore the Multiplication and the Division units). Assume our original Minor CPU design requires 2 cycles for integer functions and 4 cycles for floating point functions. In our upcoming Minor CPU, we can halve either of these latencies. 

In your report, answer: Which one should we go for? Provide statistical evidence obtained through simulations.

## Submission

1. Prepare the following files:

    a. A file named daxpy.cpp which is used for testing. This file should also include the pseudo-instructions (m5_dump_reset_stats()) as asked in [Step II](#step-ii).

    b. Any Python files you used to run your simulations (e.g., `run.py`).

    c. `stats.txt` and `config.ini` files for all the simulations, appropriately named to convey which file is from which run.

      **NOTE**: Here's a table showing all the gem5's output files you need to run and submit for this part:
    

    | #   | Step# | latency1 |   latency2  | stats.txt | config.ini |
    |-----|-------|----------|-------------|-----------|------------|
    | 1	  | i     |    NA    |   NA        |    YES    |     YES    |
    | 2	  | ii	  |    NA    |   NA        |    YES    |     YES    |
    | 3	  | iii   | opLat=1  | issLat=6    |    YES    |     YES    |
    | 4   | iii   | opLat=2  | issLat=5    |    YES    |     YES    |
    | 5	  | iii   | opLat=3  | issLat=4    |    YES    |     YES    |
    | 6	  | iii   | opLat=4  | issLat=3    |    YES    |     YES    |
    | 7	  | iii   | opLat=5  | issLat=2    |    YES    |     YES    |
    | 8	  | iii   | opLat=6  | issLat=1    |    YES    |     YES    |
    | 9	  | iv    | intLat=2 | floatLat=4  |    YES    |     YES    |
    | 10	| iv    | intLat=2 | floatLat=2  |    YES    |     YES    |
    | 11  | iv    | intLat=1 | floatLat=4  |    YES    |     YES    |


    d. The Makefile you used to compile your benchmark.


2. Additionally, separate from the above files, create a file named report.pdf that contains the answers to the above questions.

3. Submit your files to the designated sections on the Gradescope.


## Grading
The grading breakdown for this assignment is as follows:

Total Points: 100

1. **Stats files (20 points total)**: for each missing stats/config (according to the table shown above) 1 point will be reduced (20 points reduced, if none submitted).
2. **20 points for compiling daxpy.cpp file by running `make` using your Makefile.**
2. **20 for gem5 executing the run.py script.**
5. **Report (40 points)**: Each of the 4 questions is worth 10 points. Partial credit will be given for answers that do not fully answer the question.


**Warning**: read the submission instructions carefully.
Failure to adhere to the instructions will result in a loss of points.

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
