---
layout: post
title: "Convert OpenNI (*.oni) files into avi - oni2avi"
date: 2013-03-19 23:27
comments: true
categories: 
---

OpenNI is a library for work with Kinect camera. I prefer to use it instead of .net libraries from Microsoft because OpenNI is more open
and, I think, it is a more natural to use it on on unix systems.
OpenNI saves video and depth map obtained from Kinect camera into it's own data format oni. Sometimes it is desirable to have avi files instead because there are
a lot of code made for this format. So I wrote a simple command line converter from oni to avi data format available <a href="https://github.com/KirillLykov/oni2avi">oni2avi</a>. 
I developed it because one friend of mine who is using Matlab asked me to help him in converting oni file to avi. There is a plugin to Matlab which allows 
to read oni files directly but for whatever reason he could not use it. Hope, this code will be usefull for someone else.
Oni2Avi needs OpenNI, OpenCV and boost. Please, read README file. In addition, you must to use a modern C++ compiler. If you use gcc, it must be 4.6 or newer.
In case if you use Mac OS and macport, do the following:
```
sudo port install openni
sudo port install opencv
sudo port install boost
git clone git://github.com/KirillLykov/oni2avi.git
cd oni2avi
make
./oni2avi <your-file-name>.oni <your-file-name>.avi --codec=FLV1
```
If you need any support, please write to <a href="https://github.com/KirillLykov/oni2avi/issues">issues</a>, click button "New Issue" and then add a proper label - bug or question.
An alternative option is to write to my email (address is written in the About section of this blog).


__UPD__: Since I shared this tool, I've got many letters with questions. Primarily from students who are doing something with Kinect. The most questions are about building oni2avi, so below are some recomendations:

* pathes for libraries and includes in the Makefile are set to the default location for the case you are using MacOS+Macport or Ubuntu+standartPackageInstaller.
Thus if you have libs or includes in a different place, you need to specify where they are in the Makefile
* if you see a compilation error, please check that all pathes are correc. If you are on Ubuntu, also check whether you need -dev packages to have includes. Finally, check the version of libraries you are using. I suppose that OpenCV version >= 2.3, boost version >= 1.48, OpenNI version >= 1.53
* if you've found a bug, specify a version of libraries you use, command line options for the tool, and input/output files you've got.