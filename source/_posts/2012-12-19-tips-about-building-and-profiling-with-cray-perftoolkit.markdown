---
layout: post
title: "Tips about building and profiling with Cray perftoolkit"
date: 2012-12-19 16:25
comments: true
categories: 
---

If you use cray compiler, it has sense to use cray's perftoolkit for finding bottlenecks in your MPI/OpenMP application.
This post is about tips about using these tools because I always forget details. I will build and analyse LAMMPS because
it is MPI-heavy application which is portable and because I use it in my work.
Before doing anything, I would check tools available on your system and picked up the newest one.
To check available versions of cray compiler:
```
module avail cce
```
Then if the most recent version is, for example, 8.1.2, load compiler and swithc to the last version:
```
module load PrgEnv-cray
module swap cce cce/8.1.2
```
Then check and load perftools:
```
module avail perftools
module load perftools/6.0.1
```
Now, you can build you application and instrument your executable with profilers stuff:
```
make <mashine_name>
pat_build -f -u -g mpi,oi lmp_<mashine_name>
```
It will create a new executable lmp_<mashine_name>+pat. Further, you need to use this executable
for your job. If you use SLURM, write in your sbatch script something like that:
```
aprun -n 32 env PAT_RT_HWPC=1 lmp_<mashine_name>+pat < in.my
```
where -n 32 means that you use 32 nodes, the hardware performance counter experiment is defined by setting the environment
variable PAT_RT_HWPC. More info about this option can be found a the end of the following 
<a href="http://www.csc.fi/english/pages/louhi_guide/program_development/tools/craypat">page</a>>.
When you job is done, call pat_report:
```
pat_report lmp_<mashine_name>+pat+* <yourfile>.txt
```
This application will create a file with extension ap2. You can explore performance of your application in text editor
(vi  <yourfile>.txt) or using application apprentice2 which has GUI:
```
app2 <ap2 file>
```
Note, that in order to use it you need to have xwindow installed on your local mashine and when you connect to your cluster
you need to specify that it can use your monitor (-Y option):
```
ssh <username>@<mashine_name> -Y
```