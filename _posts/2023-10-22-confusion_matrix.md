---
layout: single
title:  "모델 성능 평가 - Confusion Matrix"
typora-root-url: ../
categories: Math
tag: [evaludate, 기초통계, Deep Learning]
author_profile: false
sidebar:
    nav: 'counts'
search: false
use_math: true
---

## Confusion Matrix를 사용해야 하는 이유
1. 다양한 평가 지표들$\(Recall,\, Precision,\, F_{1} Score,\, F_{\beta} Score)$를 통해서 모델이 얼마나 정확하게 분류하는지 이해할 수 있게 된다.
2. 모델이 어떤 종류의 오류를 만드는지 파악하고, 특히 FP(False Positives), FN(False Negatives)를 식별하여 모델를 향후의 개선하는데 도움을 준다.
3. 클래스의 불균형 문제에서 모델의 성능을 정확하게 평가할 수 있다. 예를 들면 총 100개의 Testset중 폐렴의 걸린 사람 5명 정상인 95명에 대해서 학습모델이 모두 건강한 사람이라고 예측해도 정확도는 $\frac{95(맞은\,개수)}{100(전체\,데이터)}$ 로 총 95%의 정확도를 가진다.