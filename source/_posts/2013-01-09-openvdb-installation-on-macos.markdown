---
layout: post
title: "OpenVDB build on MacOS"
date: 2013-01-09 18:54
comments: true
categories: 
---
<a href="http://www.openvdb.org/">OpenVDB</a> is a new library by DreamWorks which contains data structures and tools for work with three-dimensional grid.
For instance, it can be used to work with <a href="http://en.wikipedia.org/wiki/Level_set">level-sets</a>. On the openvdb web site it is written that it is checked to be build only
on RedHat Linux, so I decided to save my experience about making it on the MacOS (Lion, 10.7), I use openvdb v0-103-00. It is quite easy to do if you all optional packages are not used, 
otherwise you will have to spend some time fixing mistakes in makefile and in a test. 
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
After that you need to set up paths to libraries and includes in Makefile in the openvdb directory. I did it for my case, I would recommend to start from my Makefile.
Just apply my <a href="https://github.com/KirillLykov/code-for-blog-and-patches/blob/master/openvdb/diff-patch-macos-openvdb-0-103-0.patch">patch</a> which also 
fixes some problems in the code:
```
cd <openvdb src>
patch -p1 < <path_to_patch>/diff-patch-macos-openvdb-0-103-0.patch
```
When you are done, run `make` in the OpenVDB src directory. Then you may run tests (it takes a lot of time so I would skip it).
When I did it, I found a error (testIO) which leads to program termination. The problem is in cpp-unit I use (1.12.1), to fix it comment call of `CPPUNIT_ASSERT_NO_THROW` in the `TestCoord.cc lines 120 and 123`.


