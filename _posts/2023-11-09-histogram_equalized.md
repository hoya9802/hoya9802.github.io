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
published: true
---

## Histogram Equalized란

일반적으로 이미지는 각각의 픽셀들을 가지고 있는데 그때 해당 픽셀 값의 분포를 나타내는 그래프 표현이 Histogram(히스토그램)이고, 해당 히스토그램은 주로 어두운 픽셀값과 밝은 픽셀 값을 가지고 있습니다. 따라서 이때 이 히스토그램을 균일하게 하여 어두운 영역과 밝은 영역의 대비가 더욱 뚜렷해지면서 이전보다 더욱 뚜렷한 이미지를 얻을 수 있게 됩니다.

## 히스토그램, 정규화 히스토그램 공식

<p align="center">$h(l) =$ |{$(j,i)|f(j,i) = l$}|<br></p>
<p align="center">$\hat{h} = \frac{h(l)}{M\times N}$</p>

![input_image](/images/2023-11-09-histogram_equaized/input_image.jpeg){: .align-center}

위 그림과 같이 M과 N이 4이고 L=6인 아주 작은 영상이 있습니다. 이 영상에서 1인 화소는 7개이므로 $h(1)=7$이고, 나머지 화소의 개수를 세어보면 input image의 히스토그램은 $h = (1, 7, 4, 2, 1, 1)$이고, 이때 히스토그램 $h$를 정규화한 히스토그램의 값은 $\hat{h} = (\frac{1}{16}, \frac{7}{16}, \frac{4}{16}, \frac{2}{16}, \frac{1}{16}, \frac{1}{16}) = (0.0625, 0.4375, 0.25, 0.125, 0.0625, 0.0625)$ 으로 나오게 됩니다.

## Histogram Equalization Calculation 

|$l_{in}$|$\hat{h}(l_{in})$|$c(l_{in})$|$c(l_{in})\times5$|$l_{out}$|
|:---:|:---:|:---:|:---:|:---:|
|0|내용 2|내용 3|내용 4|dfsf|
|1|내용 6|내용 7|내용 8|
|2|내용 10|내용 11|내용 12|
|3|내용 10|내용 11|내용 12|
|4|내용 10|내용 11|내용 12|
|5|내용 10|내용 11|내용 12|
|6|내용 10|내용 11|내용 12|
