---
layout: single
title:  "[Algorithm] Merge Sort(병합 정렬)"
typora-root-url: ../
categories: Python-Algorithm
tag: [Python, Algorithm]
author_profile: false
sidebar:
    nav: 'counts'
search: true
use_math: true
redirect_from:
  - /python/algorithm
published: true
---

### Merge Sort란

Merge Sort는 리스트를 절반씩 나누어 작은 단위로 분활한 후, 각각을 정렬하면서 다시 합치는 방식의 정렬 알고리즘입니다. 리스트가 더 이상 나눌 수 없을 때까지 나눈 후, 두 개의 리스트를 비교해가며 작은 값부터 차례로 병합하여 정렬을 완성합니다. <span style="color:red">**시간 복잡도는 O($n\log{n}$)**</span>으로, 일정하게 효율적인 성능을 보장하는 알고리즘입니다.

### Merge Sort의 접근방식

Merge Sort는 분할 정복과 재귀 함수를 이용한 알고리즘입니다. 리스트를 재귀적으로 작은 단위로 나눈 후, 각각을 정렬하고, 다시 합쳐서 전체를 정렬합니다.

### 재귀함수를 이용한 Merge Sort Algorithm

아래 그림은 Merge Sort Algorithm의 과정을 나타낸 그림입니다.<br>

![MergeSort](https://drive.google.com/thumbnail?id=1wZhJiNMwz9VgI6NUlyCw9QMNtfABpx_b&sz=w1000){: .img-width-half .align-center}

분활 과정 (재귀적으로 분활 정복)
1. [4, 5, 1, 3, 2] -> 나눗셈을 이용해서 두 부분으로 나눔: [4, 5, 1]과 [3, 2]
2. [4, 5, 1] -> 같은 방식으로 나눔: [4]과 [5, 1]
3. 나머지도 같은 방식으도 더 이상 나눌 수 없는 상태가 될때까지 나눕니다.

분할 결과:
* [4], [5], [1], [3], [2] (이제 더 이상 나눌 수 없는 상태)

<br>
병합 과정 (재귀가 스택에서 빠지면서 병합)
1. [4]와 [5]을 병합 -> [4, 5]
2. [4, 5]와 [1]을 병합 -> [1, 4, 5]
3. [3]와 [2]을 병합 -> [2, 3]
4. [1, 4, 5]와 [2, 3]을 병합 -> [1, 2, 3, 4, 5]

병합 결과:
* [1, 2, 3, 4, 5]

### Python Code
<a href="https://github.com/hoya9802/Algorithm-Notes/blob/main/Algorithm/Sort/merge_sort.py">Merge Sort Algorithm 파이썬 코드입니다.</a>