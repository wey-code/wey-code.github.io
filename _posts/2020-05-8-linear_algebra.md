---
layout: post
title: "线性代数"
subtitle: "基础知识"
author: "Slc"
#header-img: "img/post-bg-dreamer.jpg"
header-mask: 0.4
tags:
  - 线性代数
---

开这个贴是因为今天上课的时候，突然发现自己线性代数完全忘得一干二净。作为大三老狗，这种基础知识还是很重要的。因此想用此贴复习一下。

# 1.矩阵特征值和特征向量

A为n阶矩阵，若数λ和n维非0列向量x满足***Ax=λx***，那么数λ称为A的特征值，x称为A的对应于特征值λ的特征向量。式Ax=λx也可写成***(A-λE)x=0***，并且\|λE-A\|叫做A的特征多项式。当特征多项式等于0的时候，称为A的***特征方程***，特征方程是一个齐次线性方程组，求解特征值的过程其实就是求解特征方程的解。

![avatar](/img/in-post/linear_algebra/1.png "特征向量定义")

![avatar](/img/in-post/linear_algebra/2.png )

实际求解的时候，可以先利用图2中的式子，求得λ。然后再根据图1的式子求得x。

