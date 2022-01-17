---
title: Single Cycle CPU Design
---

![XKCD comic](https://imgs.xkcd.com/comics/old_days.png)

{% include reading.html section="Sections 4.1-4.4" %}

{% include reading-grad.html section="Appendix C.1" %}

In this section, we will be digging into the necessary components and details on how to design a processor.
The goal of this section will be to design a full RISC-V processor, which is [Assignment 2](https://github.com/jlpteaching/dinocpu-sq20/blob/master/assignments/assignment-2.md) (Due 4/17).

## Steps to execute an instruction

"Executing" an instruction takes many different steps.
It's more complicated than just "execution."

Canonically, we have the following five steps to complete an instruction:

1. **Fetch** the instruction: get the instruction data (machine code) from memory.
2. **Decode** the instruction: figure out what we should do and read the data from the register file.
3. **Execute** the instruction: use the ALU to compute a result, the effective address, or direction of the branch.
4. **Memory**: if the instruction needs to access memory, do it.
5. **Writeback** the instruction: (if required) write the result of the instruction to the register file.

The videos below explains these steps in more detail and some example instructions.

This first video discusses the first two steps, fetching and decoding an instruction.
In this video, I also discuss some details of what a *register file* actually looks like and how it operates.

{% include video.html id="0_vnludqwx" %}

Next, we look at details of execute, memory, and writeback.

{% include video.html id="0_6hppdgq5" %}

## Data path and control path

Next, we'll discuss how to split up the hardware into two parts, the control path and the data path.
By splitting the hardware this way, we're able to use *one set of hardware* to execute many different instructions.

{% include video.html id="0_q2vuxnkt" %}

Now, we can use the following design (what you will be implementing in [DINO CPU Assignment 2]({{'modules/dino cpu/assignment2' | relative_url}})) to run *any* RISC-V instruction with *one* set of hardware!

![Single cycle DINO CPU without control wires](single-cycle-no-control-assignment-2.png)

In the next video, I go through a couple of example instructions with the datapath that you will be implementing in [DINO CPU Assignment 2]({{'modules/dino cpu/assignment2' | relative_url}}).
Note that this datapath was from WQ'21, and it varies slightly from the version in WQ'22.

{% include video.html id="1_lbwfhrfe" %}

## **QUIZ** Single cycle design

See canvas for the quiz!

## Performance and limits to a single cycle design

In these videos, we will analyze this microarchitecture to understand the performance characteristics and limits of this design.

Note that these videos will be using the design from spring quarter 2020 which is a bit different than your design.
However, the idea is the same.
The quiz below will be using *your design for Winter Quarter 2022* so you will see examples using that design too.

### Performance of the single cycle design

This video discusses the performance of the above single cycle design and provides an example of computing the performance.

{% include video.html id="0_ou13i1lw" %}

### How to improve this design and single cycle limitations

This video discusses the limitations of single cycle designs and why real systems aren't implemented this way.
In this video I introduce a second CPU design.
This design isn't very "realistic" in that no real processor implements it.
However, we have freedom to make any design that we want, and this is an interesting point in the design space.

{% include video.html id="0_qzy1eka9" %}

### **QUIZ** The Single cycle design's performance

See Canvas for the quiz!
