---
title: Teaching CompArch checklist
author: Jason Lowe-Power
---

This file explains how to get started teaching comparch.
This is mostly a set of notes from what I amd doing in Winter 2024.

## Important repositories

- DINO CPU (private): https://github.com/jlpteaching/dinocpu-private/
  - This has all of the solutions and templates for the past quarters.
  - The branch name for Winter 2024 assignment 3 solution is `wq24/updated-assginment3`
- DINO CPU (public): https://github.com/jlpteaching/dinocpu
  - This is the public version of the DINO CPU. TBH, it hasn't been updated in a while.
- gem5 for teaching: ????
  - This is a fork of gem5 that has some teaching-specific changes.
- Private repo with quizzes: https://github.com/jlpteaching/ECS154B-private
  - This repo is a mess. Only Jason can figure it out.

## Setting up assignments/classrooms

I'm using GitHub classroom to manage assignments.
Go to [https://classroom.github.com/classrooms](https://classroom.github.com/classrooms) and create a new classroom.

1. Go to https://classroom.github.com/
2. Create a new classroom
3. Choose "create new organization"
4. Name organization "<CLASS>-<QUARTER>" (e.g., ECS154B-FQ22)
5. Go to https://education.github.com/
6. Click "sign in"
7. Click "Upgrade to GitHub Team"
8. Choose the organization you created above
9. Go back to https://classroom.github.com/
10. Select your classroom
11. Click on the settings tab
12. Click "enable" for codespaces
13. Add the TAs as owners of the organization

Make sure to also add the TAs as part of the TA group in the [jlpteaching organization](https://github.com/jlpteaching/).

## Updating the comparch repo

- Update the dates and information in the `_data/course.yaml` file
- Comment out all of the assignments. You can put the following code at the top of each assignment.

```markdown
<img alt="Under construction" src="{{ "/img/under-construction.png" | relative_url }}">
Assignment coming soon

{% comment %}
...
{% endcomment %}
```

- Update the syllabus with the times for discussions.
- Update the syllabus with the times for office hours.
- Update the syllabus with the current TAs and their office hours.

## Creating assignments on gradescope

## Creating assignments on Perusall

- You can import a previous class
- Go through the current assignments and see if you want to change any papers

## Setting up Canvas

- Remove all of the unneeded links. Keep the following. **DON'T FORGET TO HIT "SAVE" AT THE BOTTOM OF THE PAGE**
  - Equitable Access
  - Piazza
  - Grades
  - Assignments
  - Gradescope
  - Quizzes
  - Perusall (201A only)
  - Syllabus
- Click on gradescope to create the courses on gradescope
  - Update the data file
- Add TA's to the course
- Modify the syllabus with "See https://jlpteaching.github.io/comparch/ for all class information."
- Choose home page -> Syllabus

### Creating the quizzes

- Update the dates in `util.py`
- Update dates on each quiz in `wq23` directory

```sh
cd Canvas\ Integration/wq23
python week2.py --course 201A
python week2.py --course 154B
python week2.py --course 154B --late
```

Adding `--late` makes it so there is an "extra" quiz that has the answers and can be completed in the 2 days after the quiz is due as specified in the syllabus.
