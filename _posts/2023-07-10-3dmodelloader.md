---
layout: post
title:  "3D 模型加载（Assimp）"
author: "daweibayu"
tags: opengl
excerpt_separator: <!--more-->
---

<!--more-->

代码完全可以参考 [AssimpAndroid](https://github.com/anandmuralidhar24/AssimpAndroid)，不过该工程年久失修，Android studio 直接导入的话会有些问题，需要指定老版本 ndk ```ndkVersion '20.0.5594570'``` 才行，
我是因为一边想学习 opengl，一边想把 C++ 捡起来，所以

## 导入三方库

### 导入 assimp

编译完后以 so 形式导入工程

### 导入 glm

[glm](https://github.com/g-truc/glm)，最新版本 [0.9.9.8](https://github.com/g-truc/glm/tree/0.9.9.8)
因为 glm 是 **header only**，所以也就没有必要编译打包了，直接源码接入就完事了

### 导入 opencv


## 