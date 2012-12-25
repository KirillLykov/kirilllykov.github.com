---
layout: post
title: "Tips about building and profiling with Cray perftoolkit"
date: 2012-12-19 16:25
comments: true
categories: 
---

Cray compiler generates <a href="http://www.nersc.gov/users/computational-systems/hopper/performance-and-optimization/compiler-comparisons/">one of the fastest code.</a> On Cray XE6 lammps compiled by Cray Compiler(8.1.2, -O2) outperform gcc code (4.7, -Ofast) in 1.6 times. If you use Cray compiler, it has sense to use Cray's perftoolkit for finding bottlenecks in your MPI/OpenMP application.
This post is about tips about using these tools because I always forget details. I will build and analyze LAMMPS.
Before doing something, check tools available on your system and pick up the newest one.
To see available versions of Cray compiler:
```
module avail cce
```
If the most recent version is, for example, 8.1.2, load compiler and switch to the last version:
```
module load PrgEnv-cray
module swap cce cce/8.1.2
```
Then check and load perftools:
```
module avail perftools
module load perftools/6.0.1
```
Now, build an application and instrument your executable with profilers stuff:
```
make <machine name>
pat_build -f -u -g mpi,oi lmp_< machine name>
```
It will create a new executable lmp_< machine _name>+pat. Further, you need to use this executable
for your job. If you use SLURM, write in your sbatch script something like that:
```
aprun -n 32 env PAT_RT_HWPC=1 lmp_< machine _name>+pat < in.my
```
where -n 32 means that you use 32 nodes, the hardware performance counter experiment is defined by setting the environment
variable PAT_RT_HWPC. More info about this option can be found a the end of the following.
<a href="http://www.csc.fi/english/pages/louhi_guide/program_development/tools/craypat">page</a>.
When you job is done, call pat_report:
```
pat_report lmp_< machine _name>+pat+* <yourfile>.txt
```
This application will create a file with extension ap2. You can explore performance of your application in text editor
(vi  <yourfile>.txt) or using application apprentice2 which has GUI:
```
app2 <ap2 file>
```
Note, that in order to use it you need to have xwindow installed on your local machine and when you connect to your cluster
you need to specify that it can use your monitor (-Y option):
```
ssh <username>@< machine _name> -Y
```