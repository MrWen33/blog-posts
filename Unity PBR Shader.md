---
title: Unity PBR Shader练习
date: 2021-04-22 23:38:47
tag: 
  - PBR
  - Unity
category: Graphic
thumbnail: 
---

使用Unity Urp练习手写PBR.
实现了单平行光光源与环境光照明, 不支持多光源与shadowmap
实现参考了LearnOpengl的PBR章节

## 要点

* 直接光
  * 漫反射: 使用Lambert模型
  * 镜面反射: 使用Cook-Torrance模型

* 间接光(环境光)
  * 漫反射: 使用Unity提供的SH光照, Lambert模型
  * 镜面反射
    * Split Sum法: 使用mipmap后的环境贴图与BRDF积分贴图

* 注意Schlick-GGX(几何遮蔽函数)中参数k的值在直接光照与IBL光照下不同
* metallic(金属度)是用于调整漫反射强度与计算金属F0值的参数, 但原始BRDF方程并不包含metallic这个参数

## 效果图

![balls](https://github.com/MrWen33/PBRExercise/blob/main/photos/balls.png?raw=true)

## 项目链接

[PBRExercise](https://github.com/MrWen33/PBRExercise)

## 相关链接

* [LearnOpengl](https://learnopengl-cn.github.io/07%20PBR/01%20Theory/) LearnOpengl PBR章节
* [Siggraph Course 2012](https://blog.selfshadow.com/publications/s2012-shading-course/) siggraph 2012 course的ppt与课程笔记