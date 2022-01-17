---
title: "Pipelined CPU Design"
---

![XKCD comic](https://imgs.xkcd.com/comics/is_it_worth_the_time.png)

We've now seen how to design a simple [single cycle](../single-cycle) CPU.
Next, we're going to start talking about how to improve its performance and look at a more realistic design.

As discussed, there are two fundamental ways for computer architects to improve performance and overcome the fundamental physical constraints of electrical systems: Parallelism and locality.
The pipelined processor leverages *parallelism*, specifically "pipelined" parallelism to improve performance and overlap instruction execution.

In the next section on [Instruction-level parallelism](../ilp), we will see another type of parallelism and how it can further increase performance.

{% include reading.html section="Sections 4.5-4.9" %}

{% include reading-grad.html section="Appendix C.2-C.7" %}

**Note:** In this section there are a couple of videos which have examples using the DINO CPU in great detail.
These are very useful for the students in 154B while they are working on the DINO CPU project.
Those of you in 201A can

## Basic pipeline design

### What is pipelining?

{% include reading.html section="Sections 4.5" %}

{% include reading-grad.html section="Appendix C.2" %}

This is a video introducing the concept of pipelining through the metaphor of laundry.
I'm sure you don't love doing laundry, but it's a pretty universal chore.
I tried using a different metaphor once... I used one of my hobbies, brewing beer, but since most people didn't know the beer brewing process, it was a bit less effective.
So, you're stuck with me talking about laundry.

{% include video.html id="0_w56q0cw5" %}

### Pipelined design for the DINO CPU

{% include reading.html section="Section 4.6" %}

{% include reading-grad.html section="Appendix C.3" %}

The videos in this section use the DINO CPU design from Spring Quarter 2020 which is slightly different than the design for this quarter.
The main difference is this quarter we have a `NextPC` unit which outputs the address of the next instruction and also whether a branch is "taken".
This is equivalent to the two muxes at the "top" of the execute stage in the previous design.

The next video explains how to modify the single-cycle DINO CPU to be pipelined and goes through an example of how a few instructions may be executed.

{% comment %}
Add video about pc+4 and tying our design to the old one.
{% endcomment  %}

{% include video.html id="0_g7shogls" %}
{% comment %}
If redo this video, then I want to use a *generic* implementation (e.g., draw one or use the book.)
{% endcomment  %}

### Example execution in pipelined DINO CPU

This video shows an example of how a program will execute in a pipelined CPU.

{% include video.html id="0_chtmmxco" %}
{% comment %}
Need to re do this video with current DINO CPU implementation? Could also do generic...
{% endcomment  %}

### Pipelined CPU performance

This video discusses the performance of a pipelined CPU.

{% include video.html id="0_zpwup7cg" %}
{% comment %}
If redo this video, then I want to use a *generic* implementation (e.g., draw one or use the book.)
I would also draw in the registers when I talk about the balanced pipeline to make it a bit more clear what is combinational logic and what is the registers
{% endcomment  %}

### **QUIZ** Basic pipelining

Use canvas to complete the quiz!

## Pipeline hazards

Now that we understand the basics of pipelining, let's look at some of the limitations and requirements to implement real system.

Note: these lectures will be useful when completing [Part II]({{'modules/dino cpu/assignment3' | relative_url}}#part-ii-implementing-forwarding) of [DINO CPU assignment 3]({{'modules/dino cpu/assignment3' | relative_url}}).

### Limits of our basic pipelined design and data hazards

{% include reading.html section="Section 4.7" %}

{% include reading-grad.html section="Appendix C.4" %}

Through the following videos we will be understanding data hazards and the way the affect the pipeline performance and implementation.
These videos are bit more detailed than some of the others.
I suggest you give yourself enough time to go through the videos a couple of times to make sure you understand these concepts.

#### Detailed examples of data hazards and dependencies

First, let's look at a very detailed example of how an example application uses this pipelined design.
This video is *very* detailed.
You may want to watch the next video first which goes over the example at a higher level and then come back to this video.

{% include video.html id="0_agprwal6" %}
{% comment %}
If redo this video, then I want to use a *generic* implementation (e.g., draw one or use the book.)
{% endcomment  %}

This video introduces data hazards and goes over a very detailed example.

Next, let's bring things up a level and look at a different way to represent the same example.

{% include video.html id="0_v82gkaut" %}

This video introduces data hazards in a slightly different way than the previous video.
After this video, you may want to watch the first video again to make sure you understand the details.

#### "Solving" data hazards with stalling

Now that we see how data hazards can cause a problem, let's look at a naive way to "solve" this problem by stalling some instructions.

{% include video.html id="0_ldm6ixoh" %}

This video discusses how to "fix" the data hazard problem by stalling instructions that cause the hazard.

> Note that in a "statically" scheduled design, the compiler can actually detect/predict which instructions are going to cause hazards and insert `nop` instructions in the right places.
> These designs are not common since you would have to recompile your program for every different hardware design.
> This is a good example of why you don't want you *micro*architecture leaking into your architecture.

### How to handle data hazards (forwarding)

This video introduces forwarding to handle data hazards.

{% include video.html id="0_l0t4w443" %}

#### Load to use hazards

This video talks about a special kind of data hazard: load to use hazards.

{% include video.html id="0_gramyi32" %}

### **QUIZ** Data hazards

Use canvas to complete the quiz!

### Control hazards and branch prediction

{% include reading.html section="Section 4.8" %}

{% include reading-grad.html section="Appendix C.2, Section 3.3" %}

This video introduces control hazards with an example.

{% include video.html id="0_w6v7pbcn" %}

This video introduces the idea of branch prediction.

{% include video.html id="0_tzm8692t" %}

This video discusses the requirements of a branch predictor and some more advanced branch predictions schemes.

{% include video.html id="0_8sbmejke" %}

### Other kinds of hazards: Structural hazards

{% include video.html id="0_9pi7eacf" %}

This video talks about structural hazards.

## Exceptions in a pipelined processor

{% include reading.html section="Section 4.9" %}

{% include video.html id="0_b0k5r233" %}

This video talks about how to handle exceptions/interrupts/errors in a pipelined processor design.

## Putting it all together: examples of pipelined execution

This video goes over an end-to-end example of the pipelined DINO CPU.
This video should be helpful on [Assignment 3.2]({{'modules/dino cpu/assignment3' | relative_url}}#part-ii-implementing-forwarding).
While it uses the design from WQ '21, there are very few differences in how the pipeline control should be implemented.
Details with the design from WQ '22 will be covered in discussion.

{% include video.html id="1_ndgicts9" %}

### **QUIZ** Pipelining review

Use canvas to complete the quiz!
