---
layout: single
title:  "Computer Vision - 히스토그램 역투영(Histogram Backprojection)"
typora-root-url: ../
categories: Computer-Vision
tag: [Histogram, Math, Computer Vision]
author_profile: false
sidebar:
    nav: 'counts'
search: true
use_math: true
redirect_from:
  - /CV/histogram_backprojection
published: false
---

## Histogram Backprojection이란

히스토그램 역투영은 이미지에서 특정 색상 또는 패턴이 어디에 있는지를 알려주는 기술입니다. 이는 주어진 색상 또는 패턴에 대한 히스토그램을 계산하고, 이 히스토그램을 사용하여 입력 이미지에서 해당 색상 또는 패턴이 있는 영역을 식별합니다. 이는 특정 객체의 특징을 잘 나타내는 색상을 기반으로 이미지를 분할하는 데 사용될 수 있습니다.

## Python Code
[Code](https://github.com/hoya9802/ComputerVision/blob/main/Histogram.ipynb)