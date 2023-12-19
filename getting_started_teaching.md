---
title: Teaching CompArch checklist
author: Jason Lowe-Power
---

This file explains how to get started teaching comparch.
This is mostly a set of notes from what I amd doing in Winter 2024.

## Important repositories

- DINO CPU (private): https://github.com/jlpteaching/dinocpu-private/
  - This has all of the solutions and templates for the past quarters.
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

## Creating the quizzes

??? Can't remember...