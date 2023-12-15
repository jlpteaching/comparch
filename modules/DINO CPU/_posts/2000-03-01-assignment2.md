---
Authors: Jason Lowe-Power, Filipe Eduardo Borges
Editor:  Hoa Nguyen
Title: DINO CPU Assignment 2
---

# DINO CPU Assignment 2

Originally from ECS 154B Lab 2, Winter 2019.

Modified for ECS 154B Lab 2, {{ site.data.course.quarter }}.

**Due on *{{ site.data.course.dates.dino_2 }}* 11:59PM PST**: See [Submission](#submission) for details

<img alt="Under construction" src="{{ "/img/under-construction.png" | relative_url }}">
Assignment coming soon

{% comment %}
# Table of Contents

* [Introduction](#introduction)
  * [Updating the DINO CPU code](#updating-the-dino-cpu-code)
  * [How this assignment is written](#how-this-assignment-is-written)
  * [Goals](#goals)
* [Single cycle CPU design](#single-cycle-cpu-design)
* [Control unit overview](#control-unit-overview)
* [Part 0: ControlTransfer unit overview](#part-0-controltransfer-unit-overview")
* [Part I: R-types](#part-i-r-types)
  * [R-type instruction details](#r-type-instruction-details)
  * [Testing the R-types](#testing-the-r-types)
* [Part II: I-types](#part-ii-i-types)
  * [I-type instruction details](#i-type-instruction-details)
  * [Testing the I-types](#testing-the-i-types)
* [Part III: Load](#part-iii-first-load-instruction)
  * [`ld` instruction details](#ld-instruction-details)
  * [Testing `ld`](#testing-ld)
* [Part IV: U-types](#part-iv-u-types)
  * [`lui` instruction details](#lui-instruction-details)
  * [`auipc` instruction details](#auipc-instruction-details)
  * [Testing the U-types](#testing-the-u-types)
* [Part V: Store](#part-v-first-store-instruction)
  * [`sd` instruction details](#sd-instruction-details)
  * [Testing `sd`](#testing-sd)
* [Part VI: Other memory instructions](#part-vi-other-memory-instructions)
  * [Other memory instruction details](#other-memory-instruction-details)
  * [Testing the other memory instructions](#testing-the-other-memory-instructions)
* [Part VII: Branch instructions](#part-vii-branch-instructions)
  * [Branch instruction details](#branch-instruction-details)
  * [Updating your ControlTransfer unit for branch instructions](#updating-your-controltransfer-unit-for-branch-instructions)
    * [Testing your ControlTransfer unit](#testing-your-controltransfer-unit)
* [Part VIII: `jal`](#part-viii-jal)
  * [`jal` instruction details](#jal-instruction-details)
  * [Testing `jal`](#testing-jal)
* [Part IX: `jalr`](#part-ix-jalr)
  * [`jalr` instruction details](#jalr-instruction-details)
  * [Testing `jalr`](#testing-jalr)
* [Part X: Full applications](#part-x-full-applications)
  * [Testing full applications](#testing-full-applications)
* [Grading](#grading)
* [Submission](#submission)
  * [Code portion](#code-portion)
  * [Academic misconduct reminder](#academic-misconduct-reminder)
  * [Checklist](#checklist)
* [Hints](#hints)
  * [`printf` debugging](#printf-debugging)

# Introduction

![Cute Dino]({{'img/dinocpu/dino-128.png' | relative_url}})

In the last assignment, you implemented the ALU control and incorporated it into the DINO CPU to test some bare-metal R-type RISC-V instructions.
In this assignment, you will implement the main control unit and ControlTransfer unit and update the ALU control unit (if needed).
After implementing the individual components and successfully passing all individual component tests, you will combine these along with the other CPU components to complete the single-cycle DINO CPU.
The simple in-order CPU design is based closely on the CPU model in Patterson and Hennessy's Computer Organization and Design.

## Updating the DINO CPU code

You should not reuse the assignment 1 template for this or other assignments.
We updated the local tests to match those on Gradescope.
Those changes are not updated in the assignment 1 templates.

If you start the assignment 2 before the assignment 1 late due date, you can reuse the code from the previous assignment by copying the modified files from the assignment.
We will update the assignment 1 repo with the solution after the assignment 1's late due date.
You can reuse the code from the solution for this assignment as well as the subsequent assignments.

### GitHub Codespaces / CSIF machines
The GitHub Classroom page for the class is located at [{{site.data.course.154b_github_classroom_link}}]({{site.data.course.154b_github_classroom_link}}).

The assignment 2 template repo is located at [{{site.data.course.154b_assignment2_github_link}}]({{site.data.course.154b_assignment2_github_link}}).

Follow the following link to access assignment 2: [{{site.data.course.154b_assignment2_invitation_link}}]({{site.data.course.154b_assignment2_invitation_link}}).

The above link will automatically create a repo in the GitHub Classroom page that only you have the access to.

In the event that the template repo is updated, your own repo won't be automatically updated.
You don't need to keep track of the template repo, unless we found an error in the assignment, in which case, we will make an announcement on {{site.data.course.discussion_site}} and provide ways to update your repo.

## How this assignment is written

The goal of this assignment is to implement a single-cycle RISC-V CPU which can execute all of the RISC-V integer instructions, i.e. RV64IM instructions.
Through the rest of this assignment, [Part I](#part-i-r-types) through [Part X](#part-x-full-applications), you will implement all of the RISC-V instructions, step by step.

If you prefer, you can simply skip to the end and implement all of the instructions at once, then run all of the tests for this assignment via the following command.
You will also use this command to test everything once you believe you're done.

```
sbt:dinocpu> Lab2 / test
```

We are making one major constraint on how you are implementing your CPU.
**You cannot modify the I/O for any module**.
We will be testing your control unit with our data path, and our data path with your control unit.
Therefore, you **must keep the exact same I/O**.
You will get errors on Gradescope (and thus no credit) if you modify the I/O.

## Goals

- Learn how to implement a control and data path in a single cycle CPU.
- Learn how different RISC-V instructions interact with the control flow of the single cycle CPU to produce appropriate data path for each type of instruction.

# Single cycle CPU design

Below is a diagram of the single cycle DINO CPU.
This diagram includes all of the necessary data path wires and muxes.
However, it is missing the control path wires.
This figure has all of the muxes necessary, but does not show which control lines go to which mux.
**Hint**: the comments in the code for the control unit give some hints on how to wire the design.

In this assignment, you will be implementing the data path shown in the figure below, implementing the control path for the DINOCPU, and wiring up the control path.
You can extend your work from [Lab 1]({{'modules/dino cpu/assignment1' | relative_url}}), or you can take the updated code from [GitHub]({{site.data.course.154b_assignment2_github_link}}).
You will be implementing everything in the diagram in Chisel (the `cpu.scala` file only implements the R-type instructions), which includes the code for the muxes.
Then, you will wire all of the components together.
You will also implement the [control unit](#control-unit-overview), ControlTransfer unit and update the ALU Control unit.

**Important Notice:**
In order to get familiar with debugging your design using single stepper, ***we strongly encourage you to watch the tutorial video*** we have provided in the link below. You may have watched it while working on assignment1. As mentioned before, the videos were originally made for spring quarter 2020 (sq20). Just in case you wanted to use any command or text from these videos which contains 'sq20', you just need to convert it to '{{site.data.course.quarter_abbr}}' to be applicable to your materials for the current quarter.

[DinoCPU - Debugging your implementation](https://video.ucdavis.edu/playlist/dedicated/0_8bwr1nkj/0_kv1v647d)

![Single cycle DINO CPU without control wires]({{'img/dinocpu/single-cycle-no-control-assignment-2.svg' | relative_url}})

# Control unit overview

In this part, you will be implementing the main control unit in the CPU design.
The control unit is used to determine how to set the control lines for the functional units and the multiplexers.

The control unit takes a single input, which is the 7-bit `opcode`.
From that input, it generates the 12 control signals listed below as output.


```scala
aluop                 Specifying the type of instruction using ALU
                          . 0 for none of the below
                          . 1 for 64-bit R-type
                          . 2 for 64-bit I-type
                          . 3 for 32-bit R-type
                          . 4 for 32-bit I-type
                          . 5 for non-arithmetic instruction types that uses ALU (auipc/jal/jarl/Load/Store)
controltransferop     Specifying the type of control transfer instruction (J-type/B-type)
                          . 0 for none of the below
                          . 1 for jal
                          . 2 for jalr
                          . 3 for branch instructions (B-type)
memop                 Specifying the type of memory instruction (Load/Store)
                          . 0 for none of the below
                          . 1 for Load
                          . 2 for Store
op1_src               Specifying the source of operand1 of ALU/ControlTransferUnit
                          . 0 if source is register file's readdata1
                          . 1 if source is pc
op2_src               Specifying the source of operand2 of ALU/ControlTransferUnit
                          . 0 if source is register file's readdata2
                          . 1 if source is a hardwired value 4
                          . 2 if source is immediate
writeback_valid       0 if not writing back to registers, 1 otherwise
writeback_src         Specifying the source of value written back to the register file
                          . 0 to select alu result
                          . 1 to select immediate generator result
                          . 2 to select data memory result
validinst             0 if the instruction is invalid, 1 otherwise
```

The following table specifies the `opcode` format and the control signals to be generated for some of the instruction types.


| opcode  | opcode format | aluop | controltransferop | memop | op1_src | op2_src | writeback_valid | writeback_src | validinst |
|---------|---------------|-------|-------------------|-------|---------|---------|-----------------|---------------|-----------|
| -       | default       |     0 |                 0 |     0 |       0 |       0 |               0 |             0 |         0 |
| 0110011 | R-type        |     1 |                 0 |     0 |       0 |       0 |               1 |             0 |         1 |
| 0111011 | R-type        |     3 |                 0 |     0 |       0 |       0 |               1 |             0 |         1 |

We have given you the control signals for the R-type instructions.
You must fill in all of the other instruction types in the table in `src/main/scala/components/control.scala`.
Notice how the third line of the table (under the `// R-type`) is an exact copy of the values in this table.

Given the input opcode, you must generate the correct control signals.
The template code from `src/main/scala/components/control.scala` is shown below.
You will fill in where it says *Your code goes here*.

```scala
// Control logic for the processor

package dinocpu.components

import chisel3._
import chisel3.util.{BitPat, ListLookup}

/**
 * Main control logic for our simple processor
 *
 * Input: opcode:                Opcode from instruction
 *
 * Output: aluop                 Specifying the type of instruction using ALU
 *                                   . 0 for none of the below
 *                                   . 1 for 64-bit R-type
 *                                   . 2 for 64-bit I-type
 *                                   . 3 for 32-bit R-type
 *                                   . 4 for 32-bit I-type
 *                                   . 5 for non-arithmetic instruction types that uses ALU (auipc/jal/jarl/Load/Store)
 * Output: controltransferop     Specifying the type of control transfer instruction (J-type/B-type)
 *                                   . 0 for none of the below
 *                                   . 1 for jal
 *                                   . 2 for jalr
 *                                   . 3 for branch instructions (B-type)
 * Output: memop                 Specifying the type of memory instruction (Load/Store)
 *                                   . 0 for none of the below
 *                                   . 1 for Load
 *                                   . 2 for Store
 * Output: op1_src               Specifying the source of operand1 of ALU/ControlTransferUnit
 *                                   . 0 if source is register file's readdata1
 *                                   . 1 if source is pc
 * Output: op2_src               Specifying the source of operand2 of ALU/ControlTransferUnit
 *                                   . 0 if source is register file's readdata2
 *                                   . 1 if source is a hardwired value 4
 *                                   . 2 if source is immediate
 * Output: writeback_valid       0 if not writing back to registers, 1 otherwise
 * Output: writeback_src         Specifying the source of value written back to the register file
 *                                   . 0 to select alu result
 *                                   . 1 to select immediate generator result
 *                                   . 2 to select data memory result
 * Output: validinst             0 if the instruction is invalid, 1 otherwise
 *
 * For more information, see section 4.4 of Patterson and Hennessy.
 * This follows figure 4.22.
 */

class Control extends Module {
  val io = IO(new Bundle {
    val opcode = Input(UInt(7.W))

    val aluop             = Output(UInt(3.W))
    val controltransferop = Output(UInt(2.W))
    val memop             = Output(UInt(2.W))
    val op1_src           = Output(UInt(1.W))
    val op2_src           = Output(UInt(2.W))
    val writeback_valid   = Output(UInt(1.W))
    val writeback_src     = Output(UInt(2.W))
    val validinst         = Output(UInt(1.W))
  })

  val signals =
    ListLookup(io.opcode,
      /*default*/           List(     0.U,               0.U,   0.U,     0.U,     0.U,             0.U,           0.U,       0.U),
      Array(              /*        aluop, controltransferop, memop, op1_src, op2_src, writeback_valid, writeback_src, validinst*/
      // R-format 64-bit operands
      BitPat("b0110011") -> List(     1.U,               0.U,   0.U,     0.U,     0.U,             1.U,           0.U,       1.U),
      // R-format 32-bit operands
      BitPat("b0111011") -> List(     3.U,               0.U,   0.U,     0.U,     0.U,             1.U,           0.U,       1.U),
      ) // Array
    ) // ListLookup

  io.aluop             := signals(0)
  io.controltransferop := signals(1)
  io.memop             := signals(2)
  io.op1_src           := signals(3)
  io.op2_src           := signals(4)
  io.writeback_valid   := signals(5)
  io.writeback_src     := signals(6)
  io.validinst         := signals(7)
}
```

In this code, you can see that the `ListLookup` looks very similar to the table above.
You will be filling in the rest of the lines of this table.
As you work through each of the parts below, you will be adding a line to the table.
You will have one line for each type of instruction (i.e., each unique opcode that for the instructions you are implementing).

You will not need to use the `validinst` signal.
It is used for exceptions and other system-related instructions that we are not implementing in this assignment.

**Important: DO NOT MODIFY THE I/O.**
You do not need to modify any other code in this file other than the `signals` table!

**Important: Any don't care lines should be set to 0!**
There may be some cases where some control signals could be 1 or 0 (i.e., you don't care what the value is).
You are *required* to set these lines to 0 for this assignment.
If you do not set these lines to 0, you will not pass the control unit test on Gradescope.


# Part 0: ControlTransfer unit overview

In the last assignment, when you were required to run your CPU for multiple cycles, you used a simple adder to generate the next value for the PC which was simply PC+4 in all cases. In this assignment you will implement a much smarter unit, TransferControlUnit, which is responsible for the next value that must be assigned to the PC for the next cycle.

The TransferControlUnit unit receives six inputs:

* `controltransferop`, which comes from the control unit.
* `operand1`, and `operand2`,  both of which come from the Register File.
* `funct3`, which comes from the instruction.
* `pc`, which is the PC.
* `imm`, which comes from the Immediate Generator unit.

The ControlTransfer unit generates two outputs:

* `nextpc`, which determines the PC of the next cycle.
* `taken`, which is True if and only if the instruction is a jump instruction, or the instruction is a branch instruction and the branch is resolved to true.

Given the inputs, you must generate the correct value for nextpc and taken. The template code from `src/main/scala/components/controltransferunit.scala` is shown below.

```scala
// Logic to calculate the next pc

package dinocpu.components

import chisel3._

/**
 * ControlTransfer Unit.
 * This component takes care of calculating/deciding the PC of the next cycle upon a control transfer instruction (jump/branch-type).
 *
 * Input: controltransferop        Specifying the type of control transfer instruction (J-type/B-type)
 *                                          . 0 for none of the below
 *                                          . 1 for jal
 *                                          . 2 for jalr
 *                                          . 3 for branch instructions (B-type)
 * Input: operand1                 First input
 * Input: operand2                 Second input
 * Input: funct3                   The funct3 from the instruction
 * Input: pc                       The *current* program counter for this instruction
 * Input: imm                      The sign-extended immediate
 *
 * Output: nextpc                  The address of the next instruction
 * Output: taken                   True if, either the instruction is a branch instruction and it is taken, or it is a jump instruction
 *
 */
class ControlTransferUnit extends Module {
  val io = IO(new Bundle {
    val controltransferop = Input(UInt(2.W))
    val operand1          = Input(UInt(64.W))
    val operand2          = Input(UInt(64.W))
    val funct3            = Input(UInt(3.W))
    val pc                = Input(UInt(64.W))
    val imm               = Input(UInt(64.W))
  
    val nextpc   = Output(UInt(64.W))
    val taken    = Output(Bool())
  })

  // default case, i.e., non-control-transfer instruction, or non-taken branch
  io.nextpc := io.pc + 4.U
  io.taken := false.B

  // Your code goes here
}
```
In this template, the outputs, `nextpc` and `taken`, are set to always be PC+4 and false, respectively.

Before starting Part I, you should remove the parts related to PC+4 and replace it with an instance of ControlTransfer unit and create proper wire connections for it.
For this purpose, you must update `src/main/scala/single-cycle/cpu.scala`.
In the next sections, you will gradually complete the body of ControlTransfer unit.

**Note**: It is fine to keep the PC+4 adder and have a mux to decide the source of the next PC.
If you choose to follow the diagram, you should remove the PC+4 adder.
As a side note, if you peek at the pipelined CPU design in assignment 3, you'll see that the PC+4 adder is required and there are muxes deciding the source of the next PC.

# Part I: R-types

In the last assignment, you implemented a subset of the RISC-V data path for just R-type instructions.
This did not require all of outputs of a control unit since there were no need for extra muxes.
However, in this assignment, you will be implementing the rest of the RISC-V instructions, and you will need to use all of outputs of the control unit (except for `validinst`).

The first step is to hook up the control unit and get the R-type instructions working again.
You shouldn't have to change all that much code in `cpu.scala` from the first assignment after you applying changes regarding [ControlTransfer unit](#part-0-controltransfer-unit-overview").
All you have to do is to hook up the `opcode` to the input of the control unit and its output `aluop` to the ALU Control Unit.
We have already implemented the R-type control logic for you.
You can also use the appropriate signals generated from the control unit (e.g., `writeback_src`) to drive your data path.

## R-type instruction details

The following table shows how an R-type instruction is laid out,

| 31-25  | 24-20 | 19-15 | 14-12   | 11-7 | 6-0   | Name   |
|--------|-------|-------|---------|------|-------|--------|
| funct7 | rs2   | rs1   | funct3  | rd   | OP    | R-type |
| funct7 | rs2   | rs1   | funct3  | rd   | OP-32 | R-type |

where `OP = 0110011` and `OP-32 = 0111011`.

Each instruction has the following effect,
- `<op>` is specified by the `funct3` and `funct7` fields.
- `R[x]` means the value stored in register x.

```
R[rd] = R[rs1] <op> R[rs2]
```

## Testing the R-types

You can run the tests for this part with the following command:

```
sbt:dinocpu> Lab2 / testOnly dinocpu.SingleCycleRTypeTesterLab2
```

# Part II: I-types

Next, you will implement the I-type instructions.
These are mostly the same as the the R-types, except that the second operand comes from the immediate value contained within the instruction, rather than another register.

**Important**: The immediate generator will produce the shifted and sign extended value!
You do not need to shift the immediate value outside of the immediate generator.

To implement the I-types, you should first extend the table in `control.scala`.
Then you can add the appropriate muxes to the CPU (in `cpu.scala`) and wire the control signals to those muxes.
**HINT**: You only need one extra mux, compared to your R-type-only design.

In this section, you will (likely) also have to update your ALU Control unit.
In assignment 1, we ignored the `aluop` input on the ALU Control unit.
Now that we are running the I-type instructions, the use of `funct7` becomes tricky.
See the table below for more information about the I-type instruction layouts.

## I-type instruction details

The following table shows how an I-type instruction is laid out,

| 31-26     | 25     | 24-20     | 19-15  | 14-12  | 11-7 | 6-0       | Name   | instructions                  |
|-----------|--------|-----------|--------|--------|------|-----------|--------|-------------------------------|
| imm[11:6] | imm[5] |  imm[4:0] |  rs1   | funct3 | rd   | OP-IMM    | I-type | ADDI/SLTI/SLTIU/ANDI/ORI/XORI |
| 000000    | imm[5] |  imm[4:0] |  rs1   | funct3 | rd   | OP-IMM    | I-type | SLLI                          |
| 000000    | imm[5] |  imm[4:0] |  rs1   | funct3 | rd   | OP-IMM    | I-type | SRLI                          |
| 010000    | imm[5] |  imm[4:0] |  rs1   | funct3 | rd   | OP-IMM    | I-type | SRAI                          |
| imm[11:6] | imm[5] |  imm[4:0] |  rs1   | funct3 | rd   | OP-IMM-32 | I-type | ADDIW                         |
| 000000    | 0      |  imm[4:0] |  rs1   | funct3 | rd   | OP-IMM-32 | I-type | SLLIW                         |
| 000000    | 0      |  imm[4:0] |  rs1   | funct3 | rd   | OP-IMM-32 | I-type | SRLIW                         |
| 010000    | 0      |  imm[4:0] |  rs1   | funct3 | rd   | OP-IMM-32 | I-type | SRAIW                         |

where `OP-IMM = 0010011`, `OP-IMM-32 = 0011011`; `imm[11:6]` indicates bit 11 to bit 6 of the immediate, and `imm[5]` indicates bit 5 of the immediate.

The funct3 values can be found in the [RISC-V ISA specs v20191213, Vol I, page 130-131](https://github.com/riscv/riscv-isa-manual/releases/download/Ratified-IMAFDQC/riscv-spec-20191213.pdf).

Note that, in general, operands in 64-bit machines are treated as 64-bit integers.
This is also true for RV64IM, except when the opcode is `OP-32` or `OP-IMM-32`, where the instructions treat the operands as 32-bit integers, and the output is signed extended from 32-bit arithmetic result to 64-bit.

Each instruction has the following effect.
`<op>` is specified by the `funct3` field.

```
R[rd] = R[rs1] <op> immediate
```

**HINT**: Due to the variation in the patterns of using `funct7` among the I-type, you can condition the output of the ALU Control unit from the `opcode` first, then `funct3`, and then `funct7` or part of `funct7`.

## Testing the I-types

You can run the tests for this part with the following command,

```
sbt:dinocpu> Lab2 / testOnly dinocpu.SingleCycleITypeTesterLab2
```

# Part III: First load instruction

Next, we will implement the `ld` instruction.
Officially, this is a I-type instruction, so you shouldn't have to make too many modifications to your data path.

As with the previous parts, first update your control unit to assert the necessary control signals for the `ld` instruction, then modify your CPU data path to add the necessary muxes and wire up your control.
For this part, you will have to think about how this instruction uses the ALU.
You will also need to incorporate the data memory into your data path, starting with this instruction.

## Data memory port I/O
The data memory port I/O is not as simple as the I/O for other modules.
It's built to be modular to allow different kinds of memories to be used with your CPU design.
We are planning to explore this further in Lab 4.
If you want to see the details, you can find them in the [mem-port-io.scala]({{site.data.course.154b_assignment2_github_link}}/blob/main/src/main/scala/memory/memory-port-io.scala) file.

The I/O for the data memory port is shown below.
Don't forget that the instruction and data memory ports look weird to use.
You have to say `io.dmem`, which seems backwards.
For this assignment, you can ignore the good and ready signals since memory will respond to the request in the same cycle.

```
Input:  address, the address of a piece of data in memory
Input:  writedata, valid interface for the data to write to the address
Input:  valid, true when the address (and writedata during a write) specified is valid
Input:  memread, true if we are reading from memory
Input:  memwrite, true if we are writing to memory
Input:  maskmode, mode to mask the result. 0 means byte, 1 means halfword, 2 means word, 3 means doubleword
Input:  sext, true if we should sign extend the result

Output: readdata, the data read and sign extended
Output: good, true when memory has the response
Output: ready, true when the memory is ready to accept another request
```

## `ld` instruction details

The following table shows how the `ld` instruction is laid out:

| 31-20     | 19-15 | 14-12 | 11-7 | 6-0     | Name   |
|-----------|-------|-------|------|---------|--------|
| imm[11:0] | rs1   | 011   | rd   | 0000011 | ld     |

`ld` stands for "load doubleword".
The instruction has the following effect,

```
R[rd] = M[R[rs1] + immediate]
```

where  `M[x]` means the value of memory at location x.

## Testing `ld`

You can run the tests for this part with the following command,

```
sbt:dinocpu> Lab2 / testOnly dinocpu.SingleCycleLoadTesterLab2
```

# Part IV: U-types

U-types are another type of instruction that look similar to the I-types.
There are two of them you need to implement, described below.

## `lui` instruction details

The following table shows how the `lui` instruction is laid out,


| 31-12      | 11-7 | 6-0     | Name   |
|------------|------|---------|--------|
| imm[31:12] | rd   | 0110111 | lui    |

`lui` stands for "load upper immediate".
The instruction has the following effect.
As in C and C++, the `<<` operator means bit shift left by the number specified.

```
R[rd] = imm << 12
```

**Important**: The immediate generator will produce the shifted and sign extended value!
You do not need to shift the immediate value outside of the immediate generator.

Use the diagram as a hint on how to modify your data path and control unit for this instruction.
There are multiple different ways to implement this instruction, so be careful to follow the diagram above!

## `auipc` instruction details

The following table shows how the `auipc` instruction is laid out.

| 31-12      | 11-7 | 6-0     | Name   |
|------------|------|---------|--------|
| imm[31:12] | rd   | 0010111 | auipc  |

`auipc` stands for "add upper immediate to pc".
The instruction has the following effect,

```
R[rd] = pc + imm << 12
```

Like previous part, use the diagram as a hint on how to modify your data path and control unit for this instruction.

## Testing the U-types

You can run the tests for this part with the following command:

```
sbt:dinocpu> Lab2 / testOnly dinocpu.SingleCycleUTypeTesterLab2
```

# Part V: First store instruction

`sd` is similar to `ld` in function, and looks similar to an I-type.
You'll need to think about how to implement the changes needed for the data memory.

## `sd` instruction details

The following table shows how the `sd` instruction is laid out.

| 31-25     | 24-20 | 19-15 | 14-12 | 11-7     | 6-0     | Name   |
|-----------|-------|-------|-------|----------|---------|--------|
| imm[11:5] | rs2   | rs1   | 011   | imm[4:0] | 0100011 | sd     |

`sd` stands for "store doubleword".
The instruction has the following effect,

```
M[R[rs1] + immediate] = R[rs2]
```

(Careful, while this looks similar to `ld`, it has a very different effect!)

## Testing `sd`

You can run the tests for this part with the following command:

```
sbt:dinocpu> Lab2 / testOnly dinocpu.SingleCycleStoreTesterLab2
```

# Part VI: Other memory instructions

We now move on to the other memory instructions.
Make sure your `ld` and `sd` instructions work before moving on to this part.

## Other memory instruction details

The following table show how the other memory instructions are laid out.
`ld` and `sd` are included again as a reference.

| 31-25     | 24-20    | 19-15 | 14-12 | 11-7     | 6-0     | Name   |
|-----------|----------|-------|-------|----------|---------|--------|
| imm[11:5] | imm[4:0] | rs1   | 000   | rd       | 0000011 | lb     |
| imm[11:5] | imm[4:0] | rs1   | 001   | rd       | 0000011 | lh     |
| imm[11:5] | imm[4:0] | rs1   | 010   | rd       | 0000011 | lw     |
| imm[11:5] | imm[4:0] | rs1   | 011   | rd       | 0000011 | ld     |
| imm[11:5] | imm[4:0] | rs1   | 100   | rd       | 0000011 | lbu    |
| imm[11:5] | imm[4:0] | rs1   | 101   | rd       | 0000011 | lhu    |
| imm[11:5] | imm[4:0] | rs1   | 110   | rd       | 0000011 | lwu    |
| imm[11:5] | rs2      | rs1   | 000   | imm[4:0] | 0100011 | sb     |
| imm[11:5] | rs2      | rs1   | 001   | imm[4:0] | 0100011 | sh     |
| imm[11:5] | rs2      | rs1   | 010   | imm[4:0] | 0100011 | sw     |
| imm[11:5] | rs2      | rs1   | 011   | imm[4:0] | 0100011 | sd     |

`l` and `s` mean "load" and "store," as mentioned previously.
`b` means a "byte" (8 bits), `h` means "half" of a word (16 bits), `w` means a "word" (32 bits), and `d` means a "double" word (64 bits).
`u` means "unsigned".

**Hint**: The data memory port has `mask` and `sext` (sign extend) inputs.
You do not need to mask or sign extend the result outside of the data memory port.
The data memory port takes care of these details for you.

The instructions have the following effects,

```
lb:  R[rd] = sext(M[R[rs1] + immediate] & 0xff)
lh:  R[rd] = sext(M[R[rs1] + immediate] & 0xffff)
lw:  R[rd] = sext(M[R[rs1] + immediate] & 0xffffffff)
ld:  R[rd] = M[R[rs1] + immediate]
lbu: R[rd] = M[R[rs1] + immediate] & 0xff
lhu: R[rd] = M[R[rs1] + immediate] & 0xffff
lwu: R[rd] = M[R[rs1] + immediate] & 0xffffffff
sb:  M[R[rs1] + immediate] = R[rs2] & 0xff
sh:  M[R[rs1] + immediate] = R[rs2] & 0xffff
sw:  M[R[rs1] + immediate] = R[rs2] & 0xffffffff
sd:  M[R[rs1] + immediate] = R[rs2]
```
where  `sext(x)` stands for "sign-extend x", and `&` stands for bit-wise AND.

## Testing the other memory instructions

You can run the tests for this part with the following command:

```
sbt:dinocpu> Lab2 / testOnly dinocpu.SingleCycleLoadStoreTesterLab2
```

# Part VII: Branch instructions

This part is a little more involved than the previous instructions.
Branch (along with jump) instructions are among the instructions that can alternate the next PC to values other than PC+4.
They are called control transfer instructions as, effectively, they alter the flow of execution.

First, you will update the ControlTransfer unit.
Then, you will update other necessary modules (e.g. control unit, ALU control if needed, etc) and then wire up the other necessary muxes.

## Branch instruction details

The following table show how the branch instructions are laid out.

| imm[12, 10:5] | rs2   | rs1   | funct3 | imm[4:1, 11] | opcode  | Name   |
|---------------|-------|-------|--------|--------------|---------|--------|
| 31-25         | 24-20 | 19-15 | 14-12  | 11-7         | 6-0     |        |
| imm[12, 10:5] | rs2   | rs1   | 000    | imm[4:1, 11] | 1100011 |  beq   |
| imm[12, 10:5] | rs2   | rs1   | 001    | imm[4:1, 11] | 1100011 |  bne   |
| imm[12, 10:5] | rs2   | rs1   | 100    | imm[4:1, 11] | 1100011 |  blt   |
| imm[12, 10:5] | rs2   | rs1   | 101    | imm[4:1, 11] | 1100011 |  bge   |
| imm[12, 10:5] | rs2   | rs1   | 110    | imm[4:1, 11] | 1100011 |  bltu  |
| imm[12, 10:5] | rs2   | rs1   | 111    | imm[4:1, 11] | 1100011 |  bgeu  |

`b` here stands for branch.
`u` again means "unsigned."
The other portion of the mnemonics stand for the operation, either:

* `eq` for equals (===)
* `ne` for not equals (=/=)
* `lt` for less than (<)
* `ge` for greater than or equal to (>=)

The instructions have the following effects,

```
if (R[rs1] <op> R[rs2])
  pc = pc + immediate
else
  pc = pc + 4
```

where `<op>` determined by `funct3` (see above).

## Updating your ControlTransfer unit for branch instructions

In this part you will be updating the ControlTransferUnit component to account for branch instructions. Similar to the CPU implementation in the book, the ControlTransferUnit will compute whether or not a branch is taken (outputting its result in `taken`, true if the branch is taken and false if the branch is not taken).

You must take the RISC-V ISA specification and implement the proper control to choose the right type of branch test.
You can find the specification in the following places:

* [the table above](#branch-instruction-details), copied from the RISC-V User-level ISA Specification v2.2, page 104
* Chapter 2 of the Specification
* Chapter 2 of the RISC-V reader
* in the front of the Computer Organization and Design book

You must now extend ControlTransfer module with additional control to generate correct value for `nextpc` and correct result for `taken`.
**HINT**: As mentioned, `funct3` wire helps differentiate between different branch instructions.
**HINT**: Use Chisel's `when` / `elsewhen` / `otherwise`, or `MuxCase` syntax.
You can also use normal operators, such as `<`, `>`, `>=`, `===`, `=/=`, etc.
**HINT**: The inputs are unsigned by default. `.asSInt` and `.asUInt` allows conversions of a chisel integer type to signed and unsigned integers respectively.

**Important Note:** In this assignment you will not use the `taken` output of ControlTransfer unit anywhere in your single cycle CPU. However, we test if your ControlTransfer unit generates correct value for it. You will utilize `taken` in the next assignment.

## Testing your ControlTransfer unit

We have updated the tests for your ControlTransfer unit. The tests, along with the other lab2 tests, are in `src/test/scala/labs/Lab2Test.scala`.

In this part of the assignment, you only need to run the ControlTransfer unit tests.
To run just these tests, you can use the sbt command `testOnly`, as demonstrated below.

```
sbt:dinocpu> testOnly dinocpu.NextPCBranchTesterLab2
```

## Implementing branch instructions

Next, you need to wire the `nextpc` output from the ControlTransfer unit into the data path.
You can follow the diagram given in [the single cycle CPU design section](#single-cycle-cpu-design).

## Testing the branch instructions

You can run the tests for the branch instructions with the following command.

```
sbt:dinocpu> Lab2 / testOnly dinocpu.SingleCycleBranchTesterLab2
```

# Part VIII: `jal`

Next, we look at the J-type instructions.
You can think of them as "unconditional jumps" as opposed to branch instructions, which are "conditional jumps".

## `jal` instruction details

The following table shows how the `jal` instruction is laid out.

| 31-12                    | 11-7 | 6-0     | Name   |
|--------------------------|------|---------|--------|
| imm[20, 10:1, 11, 19:12] | rd   | 1101111 | jal    |

`jal` stands for "jump and link."
The instruction has the following effect,

```
pc = pc + imm
R[rd] = pc + 4
```

You must properly update any required entities in your code (e.g. the control unit, ControlTransfer unit, etc). Finally, you should wire up any necessary muxes.

**HINT**: This instructions write to both PC and a register. Think of the data paths allowing both of them to be updated.

## Testing `jal`

You can run the tests for changes you made for ControlTransfer in this part with the following command,

```
sbt:dinocpu> Lab2 / testOnly dinocpu.NextPCJalTesterLab2
```

You can run the tests for this part with the following command:

```
sbt:dinocpu> Lab2 / testOnly dinocpu.SingleCycleJALTesterLab2
```

# Part IX: `jalr`

`jalr` is very similar to `jal`, with one difference.
Unlike `jal`, `jalr` has the format of an I-type instruction.

## `jalr` instruction details

The following table shows how the `jalr` instruction is laid out.

| 31-20     | 19-15 | 14-12 | 11-7 | 6-0     | Name   |
|-----------|-------|-------|------|---------|--------|
| imm[11:0] | rs1   | 000   | rd   | 1100111 | jalr   |

`jalr` stands for "jump and link register."
The instruction has the following effect,

```
pc = R[rs1] + imm
R[rd] = pc + 4
```

(Careful, there's one major difference between this and `jal`!)

You must properly update any required entities in your code (e.g. the control unit, ControlTransfer unit, etc). Finally, you should wire up any necessary muxes.

You can run the tests for changes you made for ControlTransfer unit in this part with the following command,

```
sbt:dinocpu> Lab2 / testOnly dinocpu.NextPCJalrTesterLab2
```

## Testing `jalr`

You can run the tests for this part with the following command,

```
sbt:dinocpu> Lab2 / testOnly dinocpu.SingleCycleJALRTesterLab2
```

# Testing the entire `ControlTransfer` module

Now, that you have fully implemented the ControlTransfer unit, you can run the tests for the whole unit with the following command,

```
sbt:dinocpu> Lab2 / testOnly dinocpu.NextPCTesterLab2
```

# Part X: Full applications

At this point, you should have a fully implemented RISC-V CPU!
In this final part of the assignment, you will run some full RISC-V applications.

We have provided four applications for you.

* `fibonacci`, which computes the nth Fibonacci number. The initial value of `t1` contains the Fibonacci number to compute, and after computing, the value is found in `t0`.
* `naturalsum`
* `multiplier`
* `divider`

If you have passed all of the above tests, your CPU should execute these applications with no issues!
If you do not pass a test, you may need to dig into the debug output of the test.

**Important Note:** We strongly encourage you to use [single stepper]({{ site.data.course.154b_assignment2_github_link }}/blob/main/documentation/single-stepping.md) to test your design for full applications. It will let you step through the execution one cycle at a time and print information as you go. Details on how to use the single stepper can be found in the [documentation]({{ site.data.course.154b_assignment2_github_link }}/blob/main/documentation/single-stepping.md). You can also watch the video we have provided in the link below to learn how to debug using single stepper. As mentioned earlier, the videos were originally made for spring quarter 2020 (sq20). Just in case you wanted to use any command or text from these videos which contains 'sq20', you just need to convert it to '{{ site.data.course.quarter_abbr }}' to be applicable to your materials for the current quarter.

[DinoCPU - Debugging your implementation](https://video.ucdavis.edu/playlist/dedicated/0_8bwr1nkj/0_kv1v647d)



## Testing full applications

You can run all of the applications at once with the following test,

```
sbt:dinocpu> Lab2 / testOnly dinocpu.SingleCycleApplicationsTesterLab2
```

To run a single application, you can use the following command,

```
sbt:dinocpu> Lab2 / testOnly dinocpu.SingleCycleApplicationsTesterLab2 -- -z <binary name>
```

# Grading

Grading will be done automatically on Gradescope.
See [the Submission section](#Submission) for more information on how to submit to Gradescope.

| Name                  | Percentage                 |
|-----------------------|----------------------------|
| Each instruction type | 10% each (× 9 parts = 90%) |
| Full programs         | 10%                        |

# Submission

**Warning**: read the submission instructions carefully.
Failure to adhere to the instructions will result in a loss of points.

## Code portion

You will upload the three files that you changed to Gradescope on the [Lab 2]({{ site.data.course.154b_gradescope_lab2_code_link }}) assignment.

- `src/main/scala/components/alucontrol.scala`
- `src/main/scala/components/controltransferunit.scala`
- `src/main/scala/components/control.scala`
- `src/main/scala/single-cycle/cpu.scala`

Once uploaded, Gradescope will automatically download and run your code.
This should take less than 20 minutes.
For each part of the assignment, you will receive a grade.
If all of your tests are passing locally, they should also pass on Gradescope unless you made changes to the I/O, **which you are not allowed to do**.

Note: There is no partial credit on Gradescope.
Each part is all or nothing.
Either the test passes or it fails.

## Academic misconduct reminder

You are to work on this project **individually**.
You may discuss *high level concepts* with one another (e.g., talking about the diagram), but all work must be completed on your own.

**Remember, DO NOT POST YOUR CODE PUBLICLY ON GITHUB!**
Any code found on GitHub that is not the base template you are given will be reported to SJA.
If you want to sidestep this problem entirely, don't create a public fork and instead create a private repository to store your work.
GitHub now allows everybody to create unlimited private repositories for up to three collaborators, and you shouldn't have *any* collaborators for your code in this class.

# Hints

- Start early! There is a steep learning curve for Chisel, so start early and ask questions on [{{site.data.course.discussion_site}}]({{site.data.course.discussion_link}}) and in discussion sessions.
- If you need help, come to office hours for the TA, or post your questions on [{{site.data.course.discussion_site}}]({{site.data.course.discussion_link}}).
- See [common errors]({{site.data.course.154b_assignment2_github_link}}/blob/main/documentation/common-errors.md) for some common errors and their solutions.

## Single stepper

You can also use the [single stepper]({{site.data.course.154b_assignment2_github_link}}/blob/main/documentation/single-stepping.md) to step through the execution one cycle at a time and print information as you go.
Details on how to use the single stepper can be found in the [documentation]({{site.data.course.154b_assignment2_github_link}}/blob/main/documentation/single-stepping.md).

## `printf` debugging

This is the best style of debugging for this assignment.

- Use `printf` when you want to print *during the simulation*.
  - Note: this will print *at the end of the cycle* so you'll see the values on the wires after the cycle has passed.
  - Use `printf(p"This is my text with a $var\n")` to print Chisel variables. Notice the "p" before the quote!
  - You can also put any Scala statement in the print statement (e.g., `printf(p"Output: ${io.output})`).
  - Use `println` to print during compilation in the Chisel code or during test execution in the test code. This is mostly like Java's `println`.
  - If you want to use Scala variables in the print statement, prepend the statement with an 's'. For example, `println(s"This is my cool variable: $variable")` or `println("Some math: 5 + 5 = ${5+5}")`.

{% endcomment %}