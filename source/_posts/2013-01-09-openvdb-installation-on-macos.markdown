---
layout: post
title: "OpenVDB build on MacOS"
date: 2013-02-04 18:54
comments: true
categories: 
---
<a href="http://www.openvdb.org/">OpenVDB</a> is a new library by DreamWorks which contains data structures and tools for work with three-dimensional grid.
For instance, it can be used to work with <a href="http://en.wikipedia.org/wiki/Level_set">level-sets</a>. On the openvdb web site it is written that it is checked to be build only
on RedHat Linux, so I decided to save my experience about making it on the MacOS (Lion, 10.7.5), with gcc 4.7.2 and openvdb-v0-103-1. 
First of all, you need to have <a href="http://www.macports.org/">macports</a> installed, then I would recommend to install the latest gcc.
After that install OpenVDB dependencies:
```
sudo port install boost
sudo port install tbb
sudo port install openexr
sudo port install cppunit
```
The only problem is to install optional package `glfw` <a href="http://sourceforge.net/projects/glfw/files/glfw/2.7.7/glfw-2.7.7.zip/download">(download sources)</a>:
```
unzip glfw-2.7.7.zip
cd glfw-2.7.7
make cocoa
export PREFIX=<installation path>
make cocoa-install
```
pkg-config libglfw.pc
In order to simplify linkage of openvdb with glfw (it requires opengl and cocoa), do pkg-config in glwf home:
```
sudo port install pkg-config
cd pkg-config
pkg-config libglfw.pc
export PKG_CONFIG_PATH=<path to directory containing libglfw.pc>
```
After that you need to set up paths to libraries and includes in Makefile in the openvdb directory.
I made several modifications in the openvdb, all of them can be extracted from the <a href="https://github.com/KirillLykov/openvdb-patches/blob/master/openvdb-0-103-1/openvdb-0-103-1-mac-os-10.7.5.patch">patch</a>:
- modified Makefile - added boost-system-mt library, added dependencies for vdb_view and other changes
- modified vdb_view code so it can work with OpenGL 2.1 and GLSL version 120
- modified one test which can not be compiled without a error in the gcc4.7.2

In order to apply <a href="https://github.com/KirillLykov/openvdb-patches/blob/master/openvdb-0-103-1/openvdb-0-103-1-mac-os-10.7.5.patch">patch</a>:
```
patch -p1 < openvdb-0-103-1-mac-os-10.7.5.patch
```
When you are done, run `make`, `make install` in the OpenVDB src directory. You may try to run vdb_view from the `bin` directory in the your openvdb installation path.
In order to do that, download on of the shapes from the <a href="http://www.openvdb.org/download/">openvdb site</a>, for instance <a href="http://www.openvdb.org/download/models/icosahedron.vdb.gz">icosahedron</a> and run:
```
vdb_view icosahedron.vdb
```
The result should look like on the figure below:
<center>
<img src="../../../../../images/ovdb_icosahedron.png" width="400">
</center>

Then you may run tests (it takes a lot of time so I would skip it).
When I did it, I found a error (testIO) which leads to program termination. The problem is in cpp-unit I use (1.12.1), to fix it comment call of `CPPUNIT_ASSERT_NO_THROW` in the `TestCoord.cc lines 120 and 123`.
When you will try to use your openvdb library you may have problem: dyld cannot find an image. To fix this problem you may write `export DYLD_LIBRARY_PATH=$DYLD_LIBRARY_PATH:<path_to_your_openvdb_lib>`.

