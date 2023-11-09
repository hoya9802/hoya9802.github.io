---
layout: single
title:  "Computer Vision - 히스토그램 평활화(Histogram Equalization)"
typora-root-url: ../
categories: Computer-Vision
tag: [Histogram, Math, Computer Vision]
author_profile: false
sidebar:
    nav: 'counts'
search: true
use_math: true
redirect_from:
  - /CV/histogram_equalization
published: false
---

## Histogram Equalized란

일반적으로 이미지는 각각의 픽셀들을 가지고 있는데 그때 해당 픽셀 값의 분포를 나타내는 그래프 표현이 Histogram(히스토그램)이고, 해당 히스토그램은 주로 어두운 픽셀값과 밝은 픽셀 값을 가지고 있습니다. 따라서 이때 이 히스토그램을 균일하게 하여 어두운 영역과 밝은 영역의 대비가 더욱 뚜렷해지면서 이전보다 더욱 뚜렷한 이미지를 얻을 수 있게 됩니다.

## 히스토그램 공식

<p align="center">$h(l) =$ |{$(j,i)|f(j,i) = l$}|<br></p>
<p align="center">$\hat{h} = \frac{h(l)}{M\times N}$</p>