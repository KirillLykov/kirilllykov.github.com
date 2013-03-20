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
make
./oni2avi <your-file-name>.oni <your-file-name>.avi --codec=FLV1
```
If you need any support, please write me an email which is written in the about section of this blog.