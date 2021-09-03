---
layout: project
title: "Hydrodynamics Solver: Ideal Gas and Pentane Filtration"
image: /images/2phase-filtration.png
preview: "Hydrodynamics Solver: Ideal Gas and Pentane Filtration. Written in Julia."
description: "Hydrodynamics Solver: Ideal Gas and Pentane Filtration. Written in Julia."
thumbnail: 2phase-filtration.png
category: projects
usemathjax: true
---
I'm working on this project under the supervision of Vasily Pisarev; I'm very grateful to him for his mentoring!

## About project

We simulate the flow of gas and liquid through a porous
medium with the help of Darcy's law.
$$ c = \frac{testing}{test} $$

## What I've learned so far
- Basic notions about multiphase flow and filtration:
how the flow of each phase is inhibited 
by the presence of the other phases, how Darcy's law
looks like when there are more than phase present.
- Better understanding of how to work with boundary
conditions: ghost cells, making sure they are the same
order of complexity as the rest of the scheme.
- Profiling and optimization. Type stability in Julia.
- Organization of medium size project. Function overloading,
functors, modules.

## What I would do better next time
- Start writing documentation from the start :)
- Write code in a more modular style even during early
stages.
