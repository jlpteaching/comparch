---
layout: index
published: true
title: Computer Architecture
---

## Welcome to Computer Architecture!

**UC Davis: ECS 154B and ECS 201A {{ site.data.course.quarter }}**

This website contains a set of asynchronous short-ish lecture videos for UC Davis Computer Architecture (154B/201A).
The videos were mostly taken during the pandemic (Spring '20 through Winter '22).
It is a good idea to watch the asynchronous videos before attending lecture (kind of like reading the book).
Then, during synchronous/live in-person lecture you can ask questions and get clarifications.

For more information on the class, see **[the syllabus]({{'syllabus' | relative_url}})**.

The class is broken into four main components: [Introduction to Computer Architecture]({{'modules/introduction/index/' | relative_url}}), [Processor Architecture]({{'modules/processor architecture/index/' | relative_url}}), [Memory Architecture]({{'modules/memory architecture/index/' | relative_url}}), and [Parallel Architecture]({{'modules/parallel architecture/index/' | relative_url}}).

This quarter ({{ site.data.course.quarter }}) lecture will be *in-person* in {{ site.data.course.lecture_location }}.
All lecture videos will be available on [Aggie video]({{ site.data.course.aggie_video_link }}).
I will be adding videos to the [playlist]({{ site.data.course.aggie_video_link }}) as the quarter progresses.
You are responsible for the information in both the videos and found in these pages.
All notes can be found in the [One Note notebook]({{ site.data.course.one_note_link }}).

The information here is subject to change, especially the parts later in the quarter.

## Important links

* [Syllabus]({{'syllabus' | relative_url}})
* [OneNote notebook with class notes]({{ site.data.course.one_note_link }})
* [Campuswire discussions]({{ site.data.course.discussion_link }})
* [Lecture videos]({{ site.data.course.aggie_video_link }})

## Detailed Class Outline

The class will generally be broken up into three parts, with more emphasis on the first section than the other two.
Each section will begin with the motivation for why you should care about this architectural component based on the performance or other metrics of the system.
Then, after going through the design details, we will summarize with specific example from modern systems.

Each section will have one or two project-based assignments.
Those of you in ECS 154B will be using the [DINO CPU]({{ site.data.course.dino_cpu_link }}), and those of you in ECS 201A will be using [gem5](https://www.gem5.org).
Each section wil also have an exam at the end.

### [Introduction to Computer Architecture]({{'modules/introduction/index/' | relative_url}}) (About one week)

In the first section of the class we will cover some motivation for why you should care about computer architecture and general computer architecture principles.

This first section is going to be part of the "soft launch" or "transition period" for moving to online learning.
There are due dates listed for the quizzes.
However, for this first section *there will be no late penalty*.

* [Introduction to the class]({{"modules/introduction/intro" | relative_rul }})
* [Current Computing Technology]({{"modules/introduction/technology" | relative_rul }})
* [Computer System Evaluation]({{"modules/introduction/evaluation" | relative_rul }})

### [Processor Architecture]({{"modules/processor architecture/index/" | relative_url}}) (About three weeks)

* [Instruction set architectures and RISC-V]({{"modules/processor architecture/isa/" | relative_url}}) ([DINO CPU Assignment 1]({{"modules/dino cpu/assignment1" | relative_rul }}) Due {{ site.data.course.dates.dino_1 }})
* [Single cycle CPU design]({{"/modules/processor architecture/single-cycle/" | relative_url}}) ([DINO CPU Assignment 2]({{"modules/dino cpu/assignment2" | relative_rul }}) Due {{ site.data.course.dates.dino_2 }})
* [Pipelined CPU design]({{"/modules/processor architecture/pipelined/" | relative_url}}) ([DINO CPU Assignment 3.1]({{"modules/dino cpu/assignment3" | relative_rul }}) Part 1 Due (soft) {{ site.data.course.dates.dino_31 }} & Part 2 Due {{ site.data.course.dates.dino_32 }}) (ECS 201A does not have a part 1)
* [Instruction-level parallelism]({{"modules/processor architecture/ilp/" | relative_url}})
* [Processor architecture summary]({{"modules/processor architecture/summary/" | relative_url}})

**Test on {{ site.data.course.dates.midterm }}**

### [Memory System Architecture]({{"modules/memory architecture/index/" | relative_url}}) (About three weeks)

* [Memory technology]({{"modules/memory architecture/technology/" | relative_url}})
* [Caches and memory hierarchy]({{"modules/memory architecture/caches/" | relative_url}})
* [Virtual memory]({{"modules/memory architecture/virtual/" | relative_url}})
* [Memory architecture summary]({{"modules/memory architecture/summary/" | relative_url}}) (**Assignment 4 Due {{ site.data.course.dates.dino_4 }}**)

### [Parallel Architectures]({{"modules/parallel architecture/index/" | relative_url}}) (About two weeks)

* [Parallel systems' performance]({{"modules/parallel architecture/performance/" | relative_url}})
* [Parallel architectures and programming]({{"modules/parallel architecture/architectures/" | relative_url}}) (**Assignment 5 Due {{ site.data.course.dates.dino_5 }}**)

**Final exam on {{ site.data.course.dates.final }}**

## Calendar

Calendar view available [here](https://trello.com/b/{{ site.data.course.trello_code }}/{{ site.data.course.trello_name }}/calendar).

<iframe class="trello" src="https://trello.com/b/{{ site.data.course.trello_code }}.html" height="800"></iframe>
