---
layout: post
title: "Level sets with OpenVDB. Quick intoduction. Part 1"
date: 2013-03-29 22:00
comments: true
categories: 
---
##Intro

[f_A(x,y)=x^2+1](#math-markup)

From the high-level of view <a href="http://en.wikipedia.org/wiki/Level_set">level-set method</a> (further just level set) can be considered as geometry representation, in addition to polygonal meshes and NURBS. This geometry representation simplifyes solving many problems in Computational Geometry, physical based modeling in Computer Graphics. Although, level sets was first used for geometry tracking in 80th by Osher and Sethian, the main development of this method had place in the end of 90th and during 2000th.
<!--more-->
[OpenVDB](http://www.openvdb.org/) is a new library by DreamWorks which contains data structures and tools for work with three-dimensional grids. In particular, it contains tools to work with level sets. OpenVDB is very new library, prior this library, there was only one well-developed library addressing level set - Field3D. It doesn't fit my needs for some reasons so I implemented [my own level set library](https://github.com/KirillLykov/levelset-light).

In order to use OpenVDB you need to build it. In case if you use Mac OS the following post might be useful for you - <a href="http://kirilllykov.github.com/blog/2013/02/04/openvdb-installation-on-macos/">OpenVDB build on MacOS</a>. Pay attention to the tools `vdb_print` and `vdb_view` in folder `bin` in the OpenVDB installation directory. I will use this tools for visualisation of the computation results so if you want to follow, check that they are working. In case if you are using mac, you might have problems with vdb_view because shaider language version used inside is not supported by mac. In order to fix it, have a look inside of [this patch](https://github.com/KirillLykov/OpenVDB-patches/blob/master/openvdb-0-103-1/openvdb-0-103-1-mac-os-10.7.5.patch).

##Level sets from mathematical point of view

Level set of an implicit function is a set of points where this function has a predefined value. For instance, lets get a function f(x) = x^2 - 1. It describes a parabola which intersects axe x in two points -1 and 1, thus the set of points where f is equal to 0 (a predefined value, can be any number) is {x | f(x) = 0} = {-1, 1}.

<picture with a segment>

In practice, a level set function is usually constracted as signed distance function. I.e. for a given surface and an arbitrary point, we defin such a function as a signed distance from the point to the surface. If the point is outside the surface the function value is positive, if it is inside - negative, and, finally, 0 if it is on the surface.

Coming back to the example above, the surface is composed of two points in 1D space: {-1, 1}. So all poits which belongs segment (-1, 1) are inside our surface, other - outside. We want to construct a signed distance function so, in order to start, lets get an arbitrary point 2 which is outside our surface. The distance from this point to the surface is, obviously, 1. It means that we cannot just use f(x) = x^2 - 1. For another point 0 the distance from the point to the surface is -1. As one might quess the signed distance function for our surface is f(x) = |x| - 1.

<picture with line>

1D space is not really interesting so lets move to 2D and construct a signed distance function for a circle with the center in the origin and radius R. Lets get an arbitrary point __x__ = (x, y) outside the circle (vectors are in bold here and futher). The distance from the origin to this point is basicaly | __x__|= sqrt(x^2 + y ^2). Thus the distance from the point
to the surface is | __x__| - R.

<picture with circle and point outide>

In 3D space, where  __x__=(x, y, z), the distance to the z-axis parallel cylinder surface is represented by the same function f(__x__) = sqrt(x^2 + y ^2) - R.

Following similar logic, one can write distance functions for other figures such as sphere, torus, conus, etc.

It is cool to have a new geometry representation for primitives but what if one wnats to use something more complicated? E.g. surface which constructed by surfaces of two cylinders A and B. For simplicity, cylinders are z-axe oriented, so they are described by their centers and radiuses. For cylinder A, the center and the radius are denoted by (x_A, y_A) and R_A, similar for cylinder B. Given an arbitrary point \*\*x\*\* outside interior of these figures, we have two distances from the cylinders: d_A = f_A(__x__) = sqrt((x - x_A)^2 + (y - y_A)^2) - R_A, and similar d_B. The cross-section of this two cylinder is shown below. 

<the picture with two cylinders>

Because we want to have union of this two figures, we use min(d_A, d_B) as a value of the distance function for \*\*x\*\*. In case if \*\*x\*\* is interior part, min will be applied to negative values and, thus, pick up the biggest in the module. Thus the part of the surface which is inside the the union of two of them will disappear.
  
Other set operations on level sets are represented below:   
* Union of f_1 and f_2: f_union(__x__, f~1~, f~2~) = min(f_1(__x__), f_2(__x__))
* Intersection of f_1 and f_2: f_intersect(\*\*x\*\*, f_1, f_2) = max(f_1(__x__), f_2(__x__))
* Complement of f: f_complement(__x__, f) = -f(__x__)
* Difference A \ B: f_difference(__x__, f_1, f_2) = max(f_1(__x__), -f_2(__x__))

##From continuous level sets to discrete##

Although a level set might be represented by a set of continous level set functions and operations on them, in practice, the level set geometry is stored in a file as regular grid. Every grid vertex stores the exact function value. Thus if we want to get the distance from an arbitrary pount in the space to the surface, we use interpolation: find the cell of the grid to which this point belongs to and interpolate function value in this point. For instance, in 2D case for a point (x, y) we can compute a pair of indexes (i, j) which describes the cell for the point as shown on the picture below:

<picture with interpolation>
 
##Create a level set in OpenVDB

So lets create a level set of a cylinder with center in origin and z-axis oriented. In order to do that we need to specify radius of the cylinder, the grid dimensions, and the voxel (cell of the grid) size.
Pay attention that in OpenVDB you have index space for your grid and a world space. Index space for the grid is {(i, j, k) | i,j,k<-R}, note that index might be negative. If (i, j, k) is a point in the index space, then (i * h_x, j * h_y, k * h_z) is the point in the world space, where (h_x, h_y, h_z) is a vector of voxel sizes.
The code below creates a grid which contains floats, specifies bounding box of our level set in the index space, and pass everything to makeCylinder. For simplicity uniform grid is used so h_x = h_y = h_z. In order to modify elements of the grid, accessor object is used. Inside, we have loop on all points in the grid where we compute value of the function. In order to use our grid further, we need to define interpolator which will be used with this grid. It is done by callind method `setTransform`.
It is easy to see that `makeCylinder` might be generalized by introducing additional argument of type `std::function\<float (const Vec3d&)\> lsFunc` (in C++11) which allows to fill in a grid by any signed distance level set function.
```c++
#include <openvdb/openvdb.h>

using namespace openvdb;

void makeCylinder(FloatGrid::Ptr grid, float radius, const CoordBBox& indexBB, double h)
{
  typename FloatGrid::Accessor accessor = grid->getAccessor();

  for (Int32 i = indexBB.min().x(); i <= indexBB.max().x(); ++i) {
    for (Int32 j = indexBB.min().y(); j <= indexBB.max().y(); ++j) {
      for (Int32 k = indexBB.min().z(); k <= indexBB.max().z(); ++k) {
        // transform point (i, j, k) of index space into world space
	Vec3d p(i * h, j * h, k * h);
	// compute level set function value
        float interpolatedValue = sqrt(p.x() * p.x() + p.y() * p.y()) - radius;

        accessor.setValue(Coord(i, j, k), interpolatedValue);
      }
    }
  }

  grid->setTransform(openvdb::math::Transform::createLinearTransform(h));
}

void createAndSaveCylinder()
{
  openvdb::initialize();

  openvdb::FloatGrid::Ptr grid = openvdb::FloatGrid::create(2.0);

  CoordBBox indexBB(Coord(-20, -20, -20), Coord(20, 20, 20));
  makeCylinder(grid, 5.0f, indexBB, 0.5);

  // specify dataset name
  grid->setName("LevelSetCylinder");

  // save grid in the file
  openvdb::io::File file("mygrids.vdb");
  openvdb::GridPtrVec grids;
  grids.push_back(grid);
  file.write(grids);
  file.close();
}
```
After running this application, you should get something like that:
<picture with cylinder>

##Create a narrow-band level set

It is cool to be able to create and save a regular grid in the file. Yet there is one pitfall - the file containing simple cylinder is 2.67MB in size:
```
bash-3.2$ vdb_print mygrids.vdb 
LevelSetCylinder  float -20 -20 -20  20 20 20           41x41x41  68.9KVox 2.76MB
```
As you might guess there are a lot information in the grid which we are not interested in - all grid verteces which are far away from the surface {\*\*x\*\*| f(\*\*x\*\) = 0}. OpenVDB allows to store only points which are in vicinity of the surface the code below demostrates it. For every point we save values only if the distance is between -2 to 2. In the code, `backgroundValue` specifies the module of this distance. Note, that the call of the method `signedFloodFill()` propagates the sign from initialized grid points to the uninitialize, since the background value set in the grid by module.
```c++
using namespace openvdb;
void makeCylinder(FloatGrid::Ptr grid, float radius, const CoordBBox& indexBB, double h, float backgroundValue)
{
  typename FloatGrid::Accessor accessor = grid->getAccessor();

  // outputGrid voxel sizes
  for (Int32 i = indexBB.min().x(); i <= indexBB.max().x(); ++i) {
    for (Int32 j = indexBB.min().y(); j <= indexBB.max().y(); ++j) {
      for (Int32 k = indexBB.min().z(); k <= indexBB.max().z(); ++k) {
        Vec3d p(i * h, j * h, k * h);

        float interpolatedValue = sqrt(p.x() * p.x() + p.y() * p.y() + p.z() * p.z()) - radius;

        if (fabs(interpolatedValue) < backgroundValue)
          accessor.setValue(Coord(i, j, k), interpolatedValue);
      }
    }
  }

  grid->signedFloodFill();

  grid->setTransform(openvdb::math::Transform::createLinearTransform(h));
}

void createAndWriteGrid()
{
  float backgroundValue = 2.0;
  openvdb::FloatGrid::Ptr grid = openvdb::FloatGrid::create(backgroundValue);

  CoordBBox indexBB(Coord(-20, -20, -20), Coord(20, 20, 20));
  makeCylinder(grid, 5.0f, indexBB, 0.5, backgroundValue);

  grid->setName("LevelSetSphere");

  openvdb::io::File file("mygrids.vdb");

  openvdb::GridPtrVec grids;
  grids.push_back(grid);

  file.write(grids);
  file.close();
}
```
If you run this code and use vdb_print to check out the information about grid you will get that the grid is 27x27x27 instead of 40x40x40.
For a more optimal code for sphere generation, check out `openvdb/tools/LevelSetSphere.h`.

##Read the grid
Reading the grid is straight forward, the code below demonstates it. Pay attention how to use linear interpolator in order to find a value of the level set function for an arbitrary point. Venusstatue.vdb is available [here](http://www.openvdb.org/download/models/venusstatue.vdb.zip).
```c++
#include <openvdb/openvdb.h>
#include <openvdb/tools/Interpolation.h>
#include <openvdb/tools/GridSampling.h>

void readGrid()
{
  openvdb::initialize();

  openvdb::io::File inputFile("venusstatue.vdb");
  inputFile.open();
  openvdb::GridBase::Ptr baseGrid = inputFile.readGrid("ls_venus_statue");
  inputFile.close();

  openvdb::FloatGrid::Ptr inputGrid = openvdb::gridPtrCast<openvdb::FloatGrid>(baseGrid);
  
  tools::GridSampler<openvdb::FloatTree, openvdb::tools::BoxSampler>  interpolator(inputGrid->constTree(), inputGrid->transform());
  
  Vec3d p(0.0, 0.0, 0.0); //just a point in world space

  float interpolatedValue = interpolator.wsSample(p); //ws denotes world space
}
```

In the next part I will write about more advanced things such as deforming level set surfaces using PDE.
If you have any questions, suggestions about this material or a collaboration ideas regarding level sets / openvdb, fell free to contact me.