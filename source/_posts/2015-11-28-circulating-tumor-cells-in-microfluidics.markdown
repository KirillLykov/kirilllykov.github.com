---
layout: post
title: "Circulating tumor cells in microfluidics"
date: 2015-11-28 18:30
comments: true
categories: 
---
Recently we developed software package called [uDeviceX](https://github.com/uDeviceX/uDeviceX) for blood cells flow modeling.
This software allows performing computationally efficient simulations of flow of deformable blood cells in channel geometry of arbitrary complexity.
The code is written on CUDA/C++ and is targeting GPU-enabled supercomputers such as Titan at ORNL or Piz Daint at CSCS.

We used Dissipative Particle Dynamics for the fluid modeling and fluid-structure interactions.
The solid walls were modeled using Signed Distance Function.
We employed validated red blood cell  membrane model which takes into account both elastic and viscous terms.

Although uDeviceX is not the first package where mentioned models were used, it is the most computationally efficient software and the first available online.
Our work is among the finalist of the Gordon Bell Award 2015. 
In particular, we demonstrated perfect weak scalability on the whole Titan (18,688 KX20 GPU nodes) and good strong scalability on Piz Daint.
Details about software, test cases, and performance results are published in [ACM SC'15 Proceedings](http://dl.acm.org/citation.cfm?id=2807677).

A recently published video of some simulations in microfluidics using our software:
<iframe  src="https://www.youtube.com/embed/1cYMCbct4Ow"></iframe>
The authors order corresponds to the contribution to the visualization.
[Mitsuba](https://www.mitsuba-renderer.org/)  was used for rendering.
