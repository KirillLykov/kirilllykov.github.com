---
layout: post
title: "Curvature flow in curvature space"
date: 2014-01-17 09:39
comments: true
categories: 
---
![alt text](/images/curv-flow/bunny.png "")

Recently I came across an amazing paper ["Robust fairing via Conformal Curvate flow" by K. Crane et al.](http://www.cs.columbia.edu/~keenan/Projects/ConformalWillmoreFlow/paper.pdf) at SIGGRAH 2013 and decided to reproduce the results.
The basic idea of the approach is in usage of the principal curvatures instead of vertex coordinate itself for the solution of PDE. Roughly speaking, at each iteration for every vertex the curvature is computed, than modified according to the chosen PDE, and, finally, a new position is restored out of the curvature. So for instance, you want to edit your surface or curve using Willmore flow, traditionally it is 
evaluated in terms of positions of  vertecies themself, it involves spatial dirivatives, Laplace-Beltrami operator depending on positions. Thus the implementation is complicated and the numerical scheme requires small time steps to converge. By constrast, reformulataion of the problem in terms of curvature gives a very simple numerical scheme, which works with much bigger time steps (in current work 10^8 times bigger). In addition to that, this reformulation allows to
preserve desired properties of the manifold (e.g. length, angle).

Crane et. al. created a general framework and applied it to 1D manifolds (curves) and 2D manifolds (surfaces). I reproduced results only for the 1D case, since 2D is much more time consuming - working with group of quaternions, half-densities, and dirac operator - is too much for a hobby project.

<!--more-->
Fist of all, I would recommend to read materals for the course in [Discrete Differential Geometry](http://brickisland.net/cs177fa12/?p=320) provided by K. Crane. There one might find a very detailed explanation of the length-preserving
curvature flow in the curvature space, described in the paper. In addition to that, there are formulas for the standard Willmore flow for the curve and nice scatches. You also might find some details in Crane's dissertation, but primarily if you want to implement a 3D case (there is a discretization of the Dirac operator, explanation of the exterior calculus on quaternions, many proofs used for the theorems used in the paper, etc). At the moment of the writing this post, authors of the paper didn't shared their implementation for the isometric curvature flow, so I implemented it by myself (but sometimes asking K.Crane about some details, he was very helful). 

The text will be structures as follows. First, I will give basic definitions. Than I will describe the approach by Crane et al. and show some simulation results on the models provided by K. Crane. Finally, I will briefly describe a standard Willmore flow and show simulation result for comparison.

##Definitions
Definition:  <img src="http://latex.codecogs.com/gif.latex?f:M \rightarrow \mathbb{R}^n" style="border: none; box-shadow: none;vertical-align:middle"/>, where M - (n-1)-manifold with boundary  <img src="http://latex.codecogs.com/gif.latex?\partial{M}" style="border: none; box-shadow: none;vertical-align:middle"/> is called  *immersion* if  <img src="http://latex.codecogs.com/gif.latex?D_pf:T_p M \rightarrow T_{f(p)} \mathbb{R}^n" style="border: none; box-shadow: none;vertical-align:middle"/> is injective map.


Definition: let f be an immersion of a manifold M into Euclidean space, and suppose that E is a real-valued function of f. Then a *curvature flow* is the solution to the partial differential equation
 <img src="http://latex.codecogs.com/gif.latex?\dot{f} = - \nabla{E(f)}" style="border: none; box-shadow: none;vertical-align:middle"/>. We call E *energy*.


Common choices of energies are the *Dirichlet energy*  <img src="http://latex.codecogs.com/gif.latex?E_D(f) = \frac{1}{4} \int\limits_M |\nabla f|^2 \mathrm{d}A" style="border: none; box-shadow: none;vertical-align:middle"/> and the *Willmore energy*
  <img src="http://latex.codecogs.com/gif.latex?E_w(f)=\int\limits_M H^2 \mathrm{d}A" style="border: none; box-shadow: none;vertical-align:middle"/>, where H is a mean curvature (for M without border). Further, we will work with Willmore energy. Further, Willmore energy is employed.

 
 It is easy to show that energies can be rewritten as  <img src="http://latex.codecogs.com/gif.latex?E_D=\ll\Delta f, f\gg" style="border: none; box-shadow: none;vertical-align:middle"/> and <img src="http://latex.codecogs.com/gif.latex?$E_W = \ll \Delta f, \Delta f \gg = \ll \Delta^2 f, f\gg" style="border: none; box-shadow: none;vertical-align:middle"/>, where  <img src="http://latex.codecogs.com/gif.latex?\ll,\gg" style="border: none; box-shadow: none;vertical-align:middle"/> denotes inner product in  <img src="http://latex.codecogs.com/gif.latex?L^2" style="border: none; box-shadow: none;vertical-align:middle"/>. 
 <img src="http://latex.codecogs.com/gif.latex?\Delta" style="border: none; box-shadow: none;vertical-align:middle"/> denotes Laplace-Beltrami operator, note that it itself depends on the immersion f. It leads to non-linearity of the corresponding flow equations formulated in terms of immersions.

 
In their work Crane et al. created an implementation for both 1D and 2D manifold. I will consider Willmore flow only curves. Let describe the geometry of the curve via an immersion  <img src="http://latex.codecogs.com/gif.latex?f:I \rightarrow \mathbb{R}^2, s \rightarrow f(s)" style="border: none; box-shadow: none;vertical-align:middle"/> for interval  <img src="http://latex.codecogs.com/gif.latex?I= [0, L]" style="border: none; box-shadow: none;vertical-align:middle"/>.


 We will work with two definitions of energies - one to get a Willmore flow using a standard approach  <img src="http://latex.codecogs.com/gif.latex?E_1(f)=\int\limits_0^L  k^2 \mathrm{d}s" style="border: none; box-shadow: none;vertical-align:middle"/>, another one is used for the conformal 
 Willmore flow  <img src="http://latex.codecogs.com/gif.latex?E_2(k)=\int\limits_0^L  k^2 \mathrm{d}s" style="border: none; box-shadow: none;vertical-align:middle"/>. The difference between them is in the second case it is a function of the curvature  <img src="http://latex.codecogs.com/gif.latex?k" style="border: none; box-shadow: none;vertical-align:middle"/> rather than immersion f. It allows to solve simpler PDE, having a better convergence and also restore positions in a way that preserves desired properties.

##Standard Willmore flow
First consider the solution using standard approach. Let introduce notation which will be used further:  <img src="http://latex.codecogs.com/gif.latex?T=(cos(\theta), sin(\theta))" style="border: none; box-shadow: none;vertical-align:middle"/> is a tangent vector,  <img src="http://latex.codecogs.com/gif.latex?k=\frac{\mathrm d}{\mathrm d s} \left( \theta(s) \right)" style="border: none; box-shadow: none;vertical-align:middle"/> is curvature, 
 <img src="http://latex.codecogs.com/gif.latex?\ast" style="border: none; box-shadow: none;vertical-align:middle"/> denote Hodge star,  <img src="http://latex.codecogs.com/gif.latex?e^*" style="border: none; box-shadow: none;vertical-align:middle"/> means a dual edge,  <img src="http://latex.codecogs.com/gif.latex?\phi" style="border: none; box-shadow: none;vertical-align:middle"/> is an exterior angle for the dual edge, Hodge star on primal 0-form is given by 
 <img src="http://latex.codecogs.com/gif.latex?(\ast_0)_{ii}=L_i=0.5(|f_{i+1} - f_i|+|f_i - f_{i-1}|)" style="border: none; box-shadow: none;vertical-align:middle"/>. We will use the fact that <img src="http://latex.codecogs.com/gif.latex?k=\ast \phi" style="border: none; box-shadow: none;vertical-align:middle"/> to define  <img src="http://latex.codecogs.com/gif.latex?E_1" style="border: none; box-shadow: none;vertical-align:middle"/> for the discrete curves:  <img src="http://latex.codecogs.com/gif.latex?E_1(f)=\sum \phi_i^2/L_i" style="border: none; box-shadow: none;vertical-align:middle"/>.


 Another important notation is a gradient with respect to the vertex p denoted by   <img src="http://latex.codecogs.com/gif.latex?\nabla_{p}" style="border: none; box-shadow: none;vertical-align:middle"/> - consider a triangle with a fixed basement  <img src="http://latex.codecogs.com/gif.latex?u" style="border: none; box-shadow: none;vertical-align:middle"/> and a vertex  <img src="http://latex.codecogs.com/gif.latex?p" style="border: none; box-shadow: none;vertical-align:middle"/> opposite to the basement. Without goint into details the gradient will be written as follows:

 
<img src="http://latex.codecogs.com/gif.latex?\nabla_{\gamma_{i-1}} E_i = \frac{\varphi_i}{L_i L_{i-1}} \left( \frac{v_{\perp u}}{A_i} + \frac{\varphi_i}{2L_i} \hat{u} \right)\\ 
\nabla_{\gamma_{i+1}} E_i = \frac{\varphi_i}{L_i^2} \left( \frac{u_{\perp v}-v_{\perp u}}{A_i} + \frac{\varphi_i}{2L_i} (\hat{v}-\hat{u}) \right)\\ 
\nabla_{\gamma_i} E_i = -\frac{\varphi_i}{L_i L_{i+1}} \left( \frac{u_{\perp v}}{A_i} + \frac{\varphi_i}{2L_i} \hat{v} \right)\\
" style="border: none; box-shadow: none;vertical-align:middle"/>


A scheme of the proof might be found on the DDG course page. In order to implement the flow, one shall implement these formulas. In my case these formulas didn't work as is, I added condition that if the angle is 0, than the gradient is 0, I also use absolute value of the angle. Finally, I suspect that there is a mistake somewhere in the gradient for the previous and the next vertices. However, the goal of having this method here is demonstrating how cumbersome it is and also to demonstrate that theis flow is much slower than the flow in curvature space. So since these formulas work for the square and bunny, it is enough.

At the end I would like to mention that the exterior angle should be in range <img src="http://latex.codecogs.com/gif.latex?\[-\pi, \pi\]" style="border: none; box-shadow: none;vertical-align:middle"/>.
It might be computed like that: `atan2(u.x * v.y - v.x * u.y, dot(u, v))`. Curvature for this point is then just <img src="http://latex.codecogs.com/gif.latex?\kappa_i = \phi_i / L_i" style="border: none; box-shadow: none;vertical-align:middle"/>.

##Length-preserving flow
If we consider an energy <img src="http://latex.codecogs.com/gif.latex?E_2" style="border: none; box-shadow: none;vertical-align:middle"/> instead of <img src="http://latex.codecogs.com/gif.latex?E_1" style="border: none; box-shadow: none;vertical-align:middle"/> then the resulting flow is much simpler and works for bigger timesteps. 

First give a continuum formulation. The gradient with respect to <img src="http://latex.codecogs.com/gif.latex?L^2" style="border: none; box-shadow: none;vertical-align:middle"/> inner product is <img src="http://latex.codecogs.com/gif.latex?\nabla E(k) = -2k" style="border: none; box-shadow: none;vertical-align:middle"/>,
thus the gradient flow is <img src="http://latex.codecogs.com/gif.latex?\dot{k} = -2k" style="border: none; box-shadow: none;vertical-align:middle"/>. After modifying the curvature we can restore the angle <img src="http://latex.codecogs.com/gif.latex?\theta" style="border: none; box-shadow: none;vertical-align:middle"/> by integrating the curvature <img src="http://latex.codecogs.com/gif.latex?\theta(s) = \theta_0 + \int_0^s d\theta = \theta_0 + \int_0^s \kappa\ ds" style="border: none; box-shadow: none;vertical-align:middle"/>. Having <img src="http://latex.codecogs.com/gif.latex?\theta" style="border: none; box-shadow: none;vertical-align:middle"/> we can calculate tangent vector and from that we can restore immersion <img src="http://latex.codecogs.com/gif.latex?f(s) = f_0 + \int_I T(s)\ ds" style="border: none; box-shadow: none;vertical-align:middle"/>.

The discretization is the following:


<img src="http://latex.codecogs.com/gif.latex?\kappa^{t} = \kappa^{t-1} + \tau \nabla E(\kappa^{t-1})\\
L_i = \frac{1}{2}( |\gamma_{i+1}-\gamma_i| + |\gamma_i-\gamma_{i-1}| )\\
\phi^{i}=L_{i}\kappa^{t}_i\\
\theta_i = \sum_{k=0}^i \varphi_k\\
T_i = (\cos\theta_i,\sin\theta_i)\\
\gamma_i = \sum_{k=0}^i L_k T_k
" style="border: none; box-shadow: none;vertical-align:middle"/>

In addition, we must take into account that the curve is closed, e.g. <img src="http://latex.codecogs.com/gif.latex?f(0) = f(L)" style="border: none; box-shadow: none;vertical-align:middle"/> and <img src="http://latex.codecogs.com/gif.latex?T(0) = T(L)" style="border: none; box-shadow: none;vertical-align:middle"/>. Without providing a proof - it is equavalent to having the following conditions: <img src="http://latex.codecogs.com/gif.latex?\langle \dot{\kappa}, 1 \rangle = \langle \dot{\kappa}, \gamma_x \rangle = \langle \dot{\kappa}, \gamma_y \rangle = 0" style="border: none; box-shadow: none;vertical-align:middle"/>. For the descrete case, one need to work in a space <img src="http://latex.codecogs.com/gif.latex?R^n" style="border: none; box-shadow: none;vertical-align:middle"/> where n is the number of points on the curve 
but with the inner product immersed by the function itself. It means that we need to create a diagonal mass matrix B where element on the diagonal are just <img src="http://latex.codecogs.com/gif.latex?L_i" style="border: none; box-shadow: none;vertical-align:middle"/> and when computing inner product use it
<img src="http://latex.codecogs.com/gif.latex?\langle u, v \rangle = \mathsf{u^T B v}" style="border: none; box-shadow: none;vertical-align:middle"/>. The vectors we should orthogonalize are <img src="http://latex.codecogs.com/gif.latex?{1, f^x, f^y, \dot{\kappa}}" style="border: none; box-shadow: none;vertical-align:middle"/> where <img src="http://latex.codecogs.com/gif.latex?f^x" style="border: none; box-shadow: none;vertical-align:middle"/> anf <img src="http://latex.codecogs.com/gif.latex?f^y" style="border: none; box-shadow: none;vertical-align:middle"/> are x and y components of positions. At the end, <img src="http://latex.codecogs.com/gif.latex?\dot{\kappa}" style="border: none; box-shadow: none;vertical-align:middle"/> should be orthogonal to all other vectors.

The algorithm overview is presented below.

1. Evaluate curvature <img src="http://latex.codecogs.com/gif.latex?k \leftarrow \frac{1}{2}\langle N, \Delta f\rangle" style="border: none; box-shadow: none;vertical-align:middle"/>
2. Pick a desired flow direction <img src="http://latex.codecogs.com/gif.latex?\dot{k} \leftarrow -\nabla E_2(k)" style="border: none; box-shadow: none;vertical-align:middle"/>
3. Build a constraint basis <img src="http://latex.codecogs.com/gif.latex?Orthogonalize \{1, f^x, f^y\} " style="border: none; box-shadow: none;vertical-align:middle"/>
4. Project flow onto constraints <img src="http://latex.codecogs.com/gif.latex?\dot{k} \leftarrow \dot{k} - \sum \ll\dot{k}, c_i \gg c_i " style="border: none; box-shadow: none;vertical-align:middle"/>
5. Take an explicit Euler step <img src="http://latex.codecogs.com/gif.latex?k \leftarrow k + \tau \dot{k}" style="border: none; box-shadow: none;vertical-align:middle"/>
6. Recover tangents <img src="http://latex.codecogs.com/gif.latex? T \leftarrow Integrate(k) " style="border: none; box-shadow: none;vertical-align:middle"/>
7. Recover positions <img src="http://latex.codecogs.com/gif.latex? f \leftarrow Integrate(T) " style="border: none; box-shadow: none;vertical-align:middle"/>

Note that the resulting flow is isometric by construction - we do that at two last steps.

##Simulation results and comparison
The simulation results tell us that in case of the standard approach the biggest stable time step is <img src="http://latex.codecogs.com/gif.latex?\tau <= 10^{-9}" style="border: none; box-shadow: none;vertical-align:middle"/> while for Curvature flow in curvature space we have  <img src="http://latex.codecogs.com/gif.latex?\tau < 1" style="border: none; box-shadow: none;vertical-align:middle"/>.
So it is up to <img src="http://latex.codecogs.com/gif.latex?10^{9}" style="border: none; box-shadow: none;vertical-align:middle"/> times bigger! Not suprisingly you will wait forever to obtain the flow using standard approach (several minutes) while Crane's et al. approach propagates the curve in a second or so.
Also Crane's et al. approach gives isometric flow, while standard doesn't. It might be seen on the figure below - point are distributed uniformly in the first case, and not uniformly in other:

 <img src="../../../../../images/curv-flow/crane-vs-standard.png" style="vertical-align:middle; background:white"/>

Finally, some more picture for the Isometric curvature flow:


| | | | | |
|| :-: | :-: | :-: | :-: |
|<img src="../../../../../images/curv-flow/star1.png" style="scale: 0.1">| ![alt text](../../../../../images/curv-flow/star2.png "") | ![alt text](../../../../../images/curv-flow/star3.png "") | ![alt text](../../../../../images/curv-flow/star4.png "") |

| | | | | |
|| :-: | :-: | :-: | :-: |
|<img src="../../../../../images/curv-flow/square1.png" style="scale: 0.1">| ![alt text](../../../../../images/curv-flow/square2.png "") | ![alt text](../../../../../images/curv-flow/square3.png "") | ![alt text](../../../../../images/curv-flow/square4.png "") |

And one more for the standard one:


| | | | | |
|| :-: | :-: | :-: | :-: |
|<img src="../../../../../images/curv-flow/square-w1.png" style="scale: 0.1">| ![alt text](../../../../../images/curv-flow/square-w2.png "") | ![alt text](../../../../../images/curv-flow/square-w3.png "") | ![alt text](../../../../../images/curv-flow/square-w4.png "") |




