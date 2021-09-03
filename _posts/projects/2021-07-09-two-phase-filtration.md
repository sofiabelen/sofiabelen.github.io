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

$$ \vec{v_i} = -\frac{1}{\mu_i} \hat K \cdot f_\alpha (s) \cdot \nabla P$$

where $\hat K$, the specific permeability,
$\mu$ is the dynamic viscosity, $i$ is component.

The continuity equation for each component becomes:
$$\varphi \frac{\partial \rho_i}{\partial t}
    + div (\rho_i \vec{v}_i) = 0$$ where $\rho_i = \frac{m_i}{V}$.

We use the Tait equation to relate liquid density to pressure:
$$\frac{\hat{\rho} - \rho_0}{\hat{\rho}} = C \log_{10}
    \frac{B + P}{B + P_0}$$ where $C = 0.2105$,
$\rho_0 = \frac{1}{67.28 \frac{m^3}{mol}}$, $P_0 = 0.1 MPa$,
$B = 35MPa$, in the case of $C_5H_{12}$.

Ideal gas equation of state:

$$P = \frac{RT}{M} \hat{\rho}$$

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

[Read more](https://github.com/sofiabelen/Two-Phase-Filtration/blob/main/doc/doc.pdf) or check out the [code!](https://github.com/sofiabelen/Two-Phase-Filtration)
