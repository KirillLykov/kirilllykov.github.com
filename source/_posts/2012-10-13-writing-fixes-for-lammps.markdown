---
layout: post
title: "How to write fix for LAMMPS"
date: 2012-10-13 21:08
comments: true
categories: 
---

Writing fixes is the main way of extending LAMMPS.  User can implement many things using fixes, including (but not limited):
·      changing particles characteristics (positions, velocities, forces, etc.). Example: fix_freeze.
·      reading/writing data. Example: fix_restart.
·      implementing boundary conditions. Example: fix_wall*.
·      saving information about particles for future use (previous positions, for instance).
<!--more-->
All fixes are derived from class Fix and must have constructor with the signature:
```c++
FixMine(class LAMMPS *, int, char **)
```
Every fix must be registered in LAMMPS by writing the following lines of code in the header before include guards:
```c++
#ifdef FIX_CLASS
FixStyle(name_of_your_fix_in_script,name_of_your_fix_class)
#else
```
This code allows LAMMPS to find your fix when it parses input script. In addition, your fix header must be included in the file ”style_fix.h”. In case if you use LAMMPS’ make, this file is generated automatically - all files starting with fix_ are included, so call your header the same way. Otherwise, don’t forget to add your include into ”style_fix.h”.
 
Let’s write a simple fix which will print average velocity at the end of each timestep. First of all, implement a constructor:
```c++ 
FixPrint2::FixPrint2(LAMMPS *lmp, int narg, char **arg)
: Fix(lmp, narg, arg)
{
  if (narg < 4) error->all(FLERR,"Illegal fix print command");
 
  nevery = atoi(arg[3]);
  if (nevery <= 0) error->all(FLERR,"Illegal fix print command");
}
``` 
In the constructor you may parse your fix arguments. All fixes have pretty the same syntax:
fix \<fix_identifier\> \<group_name\> \<fix_name\> \<fix_arguments\>

The first 3 parameters are parsed by LAMMPS, while <fix_arguments> should be parsed by user. In our case, we need to specify how often we want to print average velocity. For instance, once in 50 timesteps:
fix 1 print2 50

There is a special variable in Fix class called nevery which specify how often method end_of_step will be called. Thus all we need to do is just set it up.
 
The next method you need to implement is setmask:
```c++
int FixPrint2::setmask()
{
  int mask = 0;
  mask |= FixConst::END_OF_STEP;
  return mask;
}
``` 
Here user must specify which methods of your fix should be called during execution. For instance, END_OF_STEP corresponds to the end_of_step method. There are 8 most important methods:
 
initial_integrate
post_integrate
pre_exchange
pre_neighbor
pre_force
post_force
final_integrate
end_of_step
 
These methods are called in predefined order during the execution of verlet algorithm (look at the method void Verlet::run(int n) in verlet.cpp). I listed them in this order. User must understand when he want to execute his code.
 
In case if we want to write print2 fix, we need only end_of_step.
```c++
void FixPrint2::end_of_step()
{
  double** v = atom->v;
  int nlocal = atom->nlocal;
  double averageVelocity[3];
  memset(averageVelocity, 0, 3 * sizeof(double));
  for (int indexOfParticle = 0; indexOfParticle < nlocal; ++indexOfParticle) {
    MathExtra::add3(averageVelocity, v[indexOfParticle], averageVelocity);
  }
  MathExtra::scale3(1.0 / nlocal, averageVelocity);
  std::cout << averageVelocity[0] << “, ”
    << averageVelocity[1] << “, “ 
    << averageVelocity[2] << std::endl;
}
``` 
In order to use MathExtra routines, include math_extra.h. This file contains math functions to work with arrays of doubles as with math vectors.

In this code we use atom. This object is stored in the instance of Pointers class (see pointers.h). This object contains all global information about simulation system. Normally, such behaviour is achieved using Singleton design pattern but here it is implemented using using protected inheritance.

The code above computes average velocity for all particles in simulation. Yet you have one unused parameter in fix call from the script - <group_name>. This parameter specifies the group of atoms used in the fix. So we should compute average for all particles in the simulation if  group_name == all, but it can be any group. In order to use this group information, use groupbit which is defined in class Fix:
```c++
for (int indexOfParticle = 0; indexOfParticle < nlocal; ++indexOfParticle) {
  if (atom->mask[i] & groupbit) {
  //Do all job here
  }
}
```
The class Pointers contains instance of class Atom. Class atom encapsulates atoms positions, velocities, forces, etc. User can access them using particle index. Note, that particle indexes are changing every timestep because of sorting. So if you just stored position of atom from previous time step in your fix, it will not be valid on the next iteration. In order to handle this situation there are several methods which can be implemented:

  double memory_usage - return how much memory fix uses
  void grow_arrays(int) - do reallocation of the per particle arrays in your fix
  void copy_arrays(int i, int j) - copy i-th per-particle information to j-th. Used when atoms sorting is performed
  void set_arrays(int i) - sets i-th particle related information to zero 

For instance, assume you need to write a fix which will store positions of atoms from previous timestep. You will add double** x to the header file. Than add allocation code to constructor:
```
memory->create(this->x, atom->nmax, 3, "FixSavePos:x");
```
Free memory at destructor:
```
memory->destroy(x);
```
Finally, implement mentioned methods:
```c++
double FixSavePos::memory_usage()
{
  int nmax = atom->nmax;
  double bytes = 0.0;
  if (this->x != 0)
    bytes += nmax * 3 * sizeof(double);
  return bytes;
}

void FixSavePos::grow_arrays(int nmax)
{
  if (this->x != 0)
    memory->grow(this->x, nmax, 3, "FixSavePos:x");
}

void FixSavePos::copy_arrays(int i, int j)
{
  if (this->x != 0)
    memcpy(this->x[j], this->x[i], sizeof(double) * 3);
}

void FixSavePos::set_arrays(int i)
{
  if (this->x != 0)
    memset(this->x[i], 0, sizeof(double) * 3);
}
```
Now, a little bit about memory allocation: I used LAMMPS memory class which is just a bunch of template functions for allocating 1D and 2D arrays. So you need to add include “memory.h” to have access to them.