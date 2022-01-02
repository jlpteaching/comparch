---
Author: Jason Lowe-Power
Editor:  Maryam Babaie
Title: ECS 201A Assignment 1
---

# ASSIGNMENT 1 // DUE 11:59 PM SUNDAY, January 16TH 2022 (40 POINTS)

Originally from University of Wisconsin-Madison CS/ECE 752 .

Modified for ECS 201A, Winter 2022.

**Due on 01/16/2022**: See [Submission](#submission) for details

## Table of Contents

- [ASSIGNMENT 1 // DUE 11:59 PM SUNDAY, January 16TH 2022 (40 POINTS)](#assignment-1--due-1159-pm-sunday-january-16th-2022-40-points)
  - [Table of Contents](#table-of-contents)
  - [Introduction](#introduction)
  - [Step I: Compile gem5](#step-i-compile-gem5)
  - [Step II: gem5 book, Part I](#step-ii-gem5-book-part-i)
  - [Step III: Write an Interesting Application](#step-iii-write-an-interesting-application)
  - [Step IV: Using gem5](#step-iv-using-gem5)
  - [Step V: Analyzing Simulated Data](#step-v-analyzing-simulated-data)
  - [Submission](#submission)
  - [Grading](#grading)
  - [Academic misconduct reminder](#academic-misconduct-reminder)
  - [Hints](#hints)

## Introduction

You should do this assignment on your own, although you are welcome to talk to classmates in person or on Campuswire about any issues you may have encountered. The standard late assignment policy applies.

For this assignment, you will go through the Learning gem5 book you were tasked to read the first three chapters of on Wednesday, 01/11/2022. This book and most of this assignment were written and designed by Prof. Jason Lowe-Power. Specifically, for this assignment you should only need Part I of the Learning gem5 book. Although the tutorial is a work-in-progress and constantly evolving, it has been tested fairly heavily by several collaborators. Nevertheless, if you believe you've found a bug or a typo, please post on Campuswire. You can ignore the ARM and default configuration script sub-sections if you would like to, although they may be useful for you for future assignments.

## Step I: Compile gem5
Go through the Introduction and Getting Started pages of the Learning gem5 book. Make sure to get your gem5 install working before moving onto the next step -- if it doesn't work, then the next steps will not work. Furthermore, we strongly recommend you use the stable branch (v21.2), as it is the most stable public branch of gem5. We strongly suggest that you use a local Linux machine or the CSIF machines for this assignment. If you use your personal Linux machine, you can go through the entire Getting Started pages of the Learning gem5 book to prepare all the dependencies required for gem5. 

**IMPORTANT NOTE: If you decided to use the CSIF machines, please follow the steps descibed below.**

You can remotely connect to the [CSIF](http://csifdocs.cs.ucdavis.edu/) machines. The tutorial on how to use them remotely is available [here](http://csifdocs.cs.ucdavis.edu/about-us/csif-general-faq#TOC-Can-I-remotely-login-to-the-CSIF-computers-). Always use the machines with green status as shown [here](http://iceman.cs.ucdavis.edu/cgi-bin/nagios3/status.cgi?servicegroup=all&style=overview).

You do not need to use sudo to install git, gcc, and other components like the tutorial does in [Building gem5, Requirements for gem5](https://www.gem5.org/documentation/learning_gem5/part1/building/) section. Among the non-optional components required by gem5, the CSIF machines have the git, gcc, python3 installed already. You are only required to install the `SCons` for the local user by this command:
```
python3 -m pip install scons
```

On CSIF machines a new pip package will be installed in `$HOME/.local`. Thus, once you successfully installed the `SCons` package for your user, everytime you needed to compile gem5 on a CSIF machine, you should use the following command:
```
python3 $HOME/.local/bin/scons build/X86/gem5.opt -j$(nproc)
```

Note that each CSIF machine has a different number of cores. Here we recommend using `-j$(nproc)` to use the total number of cores available in the machine you are connected to. gem5 takes a long time to compile, so you should use as many threads as possible to speed up compilation! It can take upto half an hour with 8 cores or 15 mins with 16 cores on CSIF machines.

**NOTE:** There are two small issues with the compilation command in the Learning gem5 book. First, the X86 build does not compile the MinorCPU model by default. Use the following command instead:
```
python3 scons build/X86/gem5.opt -j$(nproc) CPU_MODELS=AtomicSimpleCPU,TimingSimpleCPU,O3CPU,MinorCPU
```

Second, you will need to uncomment the following system.workload line:

**NOTE:** for gem5 V21 and beyond, uncomment the following line
```
system.workload = SEWorkload.init_compatible(binary)
```

## Step II: gem5 book, Part I
For this assignment, the most important parts of the Learning gem5 book are:

* [downloading and building gem5](https://www.gem5.org/documentation/learning_gem5/part1/building/),
* [creating a simple configuration script](https://www.gem5.org/documentation/learning_gem5/part1/simple_config/),
* [how to run gem5](https://www.gem5.org/documentation/learning_gem5/part1/example_configs/),
* [adding some complexity to your first script by adding a two-level cache hierarchy](https://www.gem5.org/documentation/learning_gem5/part1/cache_config/), and
* [how to parse the gem5 output and understand the statistics](https://www.gem5.org/documentation/learning_gem5/part1/gem5_stats/).

However, you should also go through the rest of Part I because it will provide you with valuable background information on how gem5 works and this information may prove useful later in the quarter (e.g., for later homework assignments). You may also find the [YouTube recording of Part I](https://www.youtube.com/watch?v=fD3hhNnfL6k) of the Learning gem5 tutorial useful. Additionally, although the tutorial has links to the final scripts at the end of each section, it's in your best interest to walk through the tutorial step-by-step and create the scripts yourself. Finally, note that the ticks you see for your "hello world" program should be different from what the tutorial shows -- for example, we got 56435000 ticks for the hello world application running with the default two level cache file.

## Step III: Write an Interesting Application

Write a simple C++ program that implements the [Sieve of Eratosthenes](https://en.wikipedia.org/wiki/Sieve_of_Eratosthenes) and outputs one single integer at the end: the number of prime numbers <= the input argument, which you will pass in on the command line. If you are unfamiliar with this algorithm, see the above link. Compile your program as a static binary. For example, if the input is 1,000,000 the output should be: 78498.

## Step IV: Using gem5

Next, you will run your application in gem5 with the configuration script you made in the [Step II](#step-ii-gem5-book-part-i). In this step, you will change the CPU model, CPU frequency, and memory configuration while testing your sieve program. Moreover, you will also describe the changes in performance the difference configurations have.

1. Run your sieve program in gem5

    Run your sieve program in gem5 instead of the 'hello' example. You will need to change the location of the binary in your configuration file from the location of the 'hello' binary to the location of your sieve binary. Do not use se.py or any of the other, existing configuration scripts. Use the one with two levels of cache that you made in Step 2. Each run of the simulator will produce a statistics file as an output -- save the statistics files generated from each run. 

    **Warning: by default, gem5 will write the output file to the same folder (m5out) every time. Make sure to move your output file before each subsequent run.**

    **Choose an appropriate input size.** You should use something large enough that the application is interesting, but not too large that gem5 takes more than 10 minutes to execute a simulation. We found that an input size of 1,000,000 takes about 5 minutes, which is a reasonable compromise.

    **Note:** The MinorCPU (next step) takes about 10x longer than TimingSimpleCPU takes.

2. Vary the CPU model

    Change the CPU model from TimingSimpleCPU to MinorCPU.

    Hint: you may want to add a command line parameter to control the CPU model.

3. Vary the CPU Clock Frequency

    Vary the CPU clock from 1 GHz to 2 GHz to 4 GHz with both CPU models.

    Hint: again, you may want to add a command line parameter for the frequency.

4. Vary the Main Memory

    The default memory configuration in your script is DDR3_1600_8x8. Change the memory configuration so you gather results with four different main memory configurations:

    * DDR3_1600_8x8 (the default), which models DDR3 memory.
    * DDR3_2133_8x8, which models DDR3 with a faster clock.
    * LPDDR2_S4_1066_1x32, which models LPDDR2, a low-power DRAM often found in mobile devices.
    * HBM_1000_4H_1x64, which models High Bandwidth Memory, a high performance memory often used in GPUs and network devices.

    Run each of these memory configurations with both MinorCPU and TimingSimpleCPU. Leave the frequency fixed at 4 GHz for this step.

5. Verify Collected Data

    You should have twelve statistics files from your runs in the above steps:

    | CPU Model         | CPU Frequency (GHz)   | Memory              |
    |-------------------|-----------------------|---------------------|
    | TimingSimpleCPU	  | 1                     | DDR3_1600_8x8       |       
    | TimingSimpleCPU	  | 2	                    | DDR3_1600_8x8       |
    | TimingSimpleCPU	  | 4	                    | DDR3_1600_8x8       |
    | MinorCPU          | 1	                    | DDR3_1600_8x8       |
    | MinorCPU	        | 2	                    | DDR3_1600_8x8       |
    | MinorCPU	        | 4	                    | DDR3_1600_8x8       |
    | TimingSimpleCPU	  | 4	                    | DDR3_2133_8x8       |
    | TimingSimpleCPU	  | 4	                    | LPDDR2_S4_1066_1x32 |
    | TimingSimpleCPU	  | 4	                    | HBM_1000_4H_1x64    |
    | MinorCPU	        | 4	                    | DDR3_2133_8x8       |
    | MinorCPU	        | 4	                    | LPDDR2_S4_1066_1x32 |
    | MinorCPU	        | 4	                    | HBM_1000_4H_1x64    |


## Step V: Analyzing Simulated Data
After collecting all of the data from the previous step, analyze the statistics your runs generated and write a report. Your report answering should be a PDF, and should answer the following questions:

1. What metric should you use to compare the performance between different system configurations? Why is this the appropriate metric?
2. Which CPU model is more sensitive to changing the CPU frequency? Why?
3. Which CPU model is more sensitive to changing the memory technology? Why?
4. Is your sieve application more sensitive to the CPU model, the memory technology, or CPU frequency? Why?
5. If you were to use a different application, do you think your conclusions would change? Why?


## Submission
1. Create an archive (.zip, .gz, or .tgz) of the following files:
    a. A file named sieve.cpp with your implementation of the Sieve of Eratosthenes.
    b. A file named sieve-config.py (and any other necessary files) that was used to run gem5.
    c. All 12 statistics files (stats.txt) from your runs of your sieve program in gem5, appropriately named to convey which stats file is from which run.

2. Additionally, separate from the above archive, create a file named report.pdf that contains a short report with your observations and conclusions from the experiment, including answers to the above questions.

3. Submit your archive and report on Canvas.


## Grading
The grading breakdown for this assignment is as follows:

Total Points: 40

1. **Stats files (12 points total):** Each stats file is worth 1 point if it is submitted, 0 otherwise.
2. **sieve.cpp (4 points)**
3. **sieve-config.py (4 points)**
4. **Report (20 points):** Each of the 5 questions is worth 4 points. Partial credit will be given for answers that do not fully answer the question.


**Warning**: read the submission instructions carefully.
Failure to adhere to the instructions will result in a loss of points.


## Academic misconduct reminder

You are to work on this project **individually**.
You may discuss *high level concepts* with one another (e.g., talking about the sieve algorithm), but all work must be completed on your own.

**Remember, DO NOT POST YOUR CODE PUBLICLY ON GITHUB!**
Any code found on GitHub that is not the base template you are given will be reported to SJA.
If you want to sidestep this problem entirely, don't create a public fork and instead create a private repository to store your work.
GitHub now allows everybody to create unlimited private repositories for up to three collaborators, and **you shouldn't have *any* collaborators** for your code in this class.

## Hints

* Start early! There is a learning curve for gem5, so start early and ask questions on Campuswire and in discussion.
* If you need help, come to office hours for the TA, or post your questions on Campuswire.

