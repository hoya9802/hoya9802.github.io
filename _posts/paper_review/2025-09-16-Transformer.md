---
layout: single
title:  "Paper Review - Transformer(Attention is all you need)"
typora-root-url: ../
categories: Paper-Review
tag: [Transformer, Attention, RNN]
author_profile: false
sidebar:
    nav: 'counts'
search: true
use_math: true
liquid: false
redirect_from:
  - /paper/Transformer
published: true
---

**[Reference]** [NIPS2017 Transformer Paper](https://arxiv.org/pdf/1706.03762){:target="_blank"}
{: .notice}

## Abstract

2017년 Google이 발표한 "Attention Is All You Need" 논문은 자연어 처리 분야에 혁신을 가져왔습니다. 기존의 RNN과 CNN을 완전히 배제하고 <span style='color:red'>오직 어텐션 메커니즘만을 사용한 Transformer 아키텍처</span>는 현재 ChatGPT, BERT 등 주요 언어 모델의 기반이 되었습니다.


## 1. Introduction

![RNN](https://drive.google.com/thumbnail?id=103ZXR7XNl7BJJUG_m3-oW0PLWSdMIi9n&sz=w1000){: .align-center}

### RNN의 구조적 한계

**1. 순차 처리의 병목**

- $x_{t}$를 처리하기 위해서는 $x_{t-1}$이 완료되어야 함

**2. 장거리 의존성 학습의 어려움**
- 그라디언트 소실 문제로 인해 먼 거리의 단어 간 관계 학습이 어려움
- LSTM, GRU도 근본적 해결책은 되지 못함

**3. 훈련 속도 제약**
- 순차 처리로 인해 병렬화가 불가능
- GPU의 병렬 연산 능력을 충분히 활용하지 못함
- 긴 시퀀스일수록 훈련 시간이 선형적으로 증가

Transformer는 이러한 문제를 **Self-Attention** 메커니즘으로 해결했습니다.

## 2. Background

순차 처리의 한계를 극복하기 위해 [ConvS2S](https://arxiv.org/pdf/1705.03122){:target="_blank"}나 [ByteNet](https://arxiv.org/pdf/2410.20855){:target="_blank"} 같은 모델들이 등장했습니다. 이들은 컨볼루션(Convolution) 연산을 이용해 병렬 처리를 시도했습니다. 하지만 이들 역시 입력값 사이의 거리가 멀어질수록 연산량이 늘어나, 장거리 의존성 문제를 완전히 해결하지는 못했습니다.

Self-Attention의 장점

- 어떤 두 단어든 한 번의 연산으로 직접 연결
- 모든 위치를 동시에 병렬 처리
- 정보 손실 없이 원본 관계 보존

기존에도 어텐션 메커니즘은 존재했지만, Transformer는 최초로:

- 순환 구조 완전 제거: RNN/LSTM 없이 순수 어텐션만 사용
- 컨볼루션 없이 구현: CNN 레이어 대신 Self-Attention으로 대체
- End-to-End 어텐션: 입력부터 출력까지 전 과정에서 어텐션만 활용

## 3. Model Architecture

![Transformer](https://drive.google.com/thumbnail?id=1CNXvTXBuAmAn6SpTtimklDMZTCmjbBeN&sz=w1000){: .img-width-half .align-center}

Transformer는 전통적인 encoder-decoder 구조를 따르며, 각각 N개의 동일한 레이어를 쌓아올린 형태입니다 (논문에서는 N=6). 핵심 특징은 self-attention과 point-wise feed-forward 네트워크를 조합하여 사용한다는 점입니다.

### 3.1. Encoder and Decoder Stacks

**Encoder**
- 6개의 동일한 레이어로 구성
- 각 레이어는 2개의 서브레이어를 포함:
  - **Multi-head self-attention mechanism**
  - **Position-wise fully connected feed-forward network**
- 각 서브레이어 주위에 residual connection 적용 후 layer normalization
- 출력: $LayerNorm(x + Sublayer(x))$
- 모든 서브레이어와 임베딩 레이어는 동일한 차원 $d_{model} = 512$ 사용

Position-wise fully connected feed-forward network: 각 토큰(position)을 개별적으로 MLP에 통과
{: .notice--info}

**Decoder**
- 마찬가지로 6개의 동일한 레이어로 구성
- Encoder의 2개 서브레이어에 추가적인 서브레이어 포함:
  - **Masked multi-head self-attention**
  - **Encoder-decoder attention** 추가
  - **positionwise fully connected feed-forward network**
- 각 서브레이어 주위에 residual connection 적용 후 layer normalization
- 출력: $LayerNorm(x + Sublayer(x))$
- 모든 서브레이어와 임베딩 레이어는 동일한 차원 $d_{model} = 512$ 사용

Decoder에서는 self-attention에서 masking으로 auto-regressive 속성 보장 (i번째 위치는 i-1까지의 정보만 활용)
{: .notice--info}

### 3.2. Attention

![Attention](https://drive.google.com/thumbnail?id=18pH_V868Sn0_3u6aaS0pFwFMh-6zVCPe&sz=w1000){: .align-center}

#### 3.2.1. Scaled Dot-Product Attention

Transformer의 핵심인 attention 함수는 query, key-value 쌍들을 output으로 매핑합니다.

<p align="center">
$Attention(Q, K, V) = softmax(\frac{QK^T}{√d_k})V$
</p>

- **Q (queries)**: "내가 지금 어떤 정보를 찾고 싶다"라는 질문을 담은 벡터(dimension $d_{k}$)
- **K (keys)**: "내가 가진 정보의 특징은 이런 거다"라고 설명하는 벡터(dimension $d_{k}$)
- **V (values)**: 실제로 전달할 정보(콘텐츠) 벡터(dimension $d_{v}$)
- **스케일링 팩터 $\frac{1}{√d_k}$**: 큰 $d_{k}$에서 gradient vanishing 문제 해결

#### 3.2.2. Multi-Head Attention

단일 attention 함수 대신, 서로 다른 learned linear projection을 통해 h번 parallel하게 attention을 수행:

<div>
$$
\begin{aligned}
MultiHead(Q, K, V)&=Concat(head_{1}, ..., head_{h})W^{O}\\
\text{where } head_{i}&=Attention(QW_{i}^{Q},KW_{i}^{k},VW_{i}^{v})
\end{aligned}
$$
</div>

- **$h = 8$**: parallel attention layer (head) 개수
- **$d_{k} = d_{v} = d_{model}/h = 64$**: 각 head의 차원
- 다양한 representation subspace의 정보를 동시에 처리 가능(집단 지성)

각 head의 차원을 줄여 사용하기 때문에 전체 연산 비용은 single-head attention과 비슷합니다.
{: .notice--info}

#### 3.2.3. Applications of Attention in the Model

1. **Encoder-Decoder Attention**: Decoder의 query가 Encoder의 key, value에 attention
2. **Encoder Self-Attention**: Encoder 내에서 모든 위치가 서로 attention
3. **Masked Decoder Self-Attention**: 
  - Decoder에서 현재 위치까지만 attention(auto-regressive)
  - 마스킹할 부분은 $-\infty$로 설정(softmax을 통과하기 때문)

### 3.3. Position-wise Feed-Forward Networks

각 레이어는 attention 외에도 position별로 독립적으로 적용되는 FFN을 포함:

<div>
$$
FFN(x) = max(0, xW_{1} + b_{1})W_{2} + b_{2}
$$
</div>

- **입출력 차원**: $d_{model}$ = 512
- **내부 차원**: $d_{ff}$ = 2048(어느정도 정보손실 완화)
- **활성화 함수**: ReLU

Position-wise FFN은 각 토큰 위치별로 독립적으로 적용되는 두 층의 Linear 변환(ReLU 포함)으로, 2번의 1x1 Conv1D와 수학적으로 동일합니다.
{: .notice--info}

### 3.4. Embeddings and Softmax

 - Transformer는 입력·출력 토큰을 $d_{model}$차원의 임베딩으로 변환하고, 디코더 출력은 Linear + Softmax로 다음 토큰 확률을 예측한다.

 - 두 임베딩 레이어와 pre-softmax linear transformation은 동일한 가중치 행렬을 공유한다.

 - 임베딩 레이어에서는 값에 $\sqrt{d_{model}}$을 곱해 스케일을 조정한다.

Pre-softmax linear transformation : 디코더 출력(hidden state)을 어휘(vocab) 크기만큼의 점수(logit)로 변환하는 선형층<br>
Logit : Log-odds의 줄임말로, 확률을 계산하기 위한 중간 단계의 점수(raw score)
{: .notice--info}

### 3.5. Positional Encoding

![PE](https://drive.google.com/thumbnail?id=1f0Cbkg77MA1pG-Dva_8ZjMDoxtK3sq75&sz=w1000){: .align-center}

Transformer는 recurrence나 convolution이 없으므로 위치 정보를 명시적으로 주입하기 위해 사진과 같이 위치별로 다른, 고정된 벡터를 사용합니다.

 - 논문에서는 학습된 방식(learned)과 고정된 방식(fixed) 중 고정된 방식을 사용
 - 고정된 오프셋을 통해 상대적 위치 정보 학습에 유리하다고 판단
 - 학습형 임베딩과 비교했을 때 성능 차이는 거의 없음

논문에서 Table 3의 base 모델(fixed)과 E 모델(learned)의 성능차이가 없음
{: .notice--info}

## 4. Why Self-Attention

![ComplexityPerLayer](https://drive.google.com/thumbnail?id=128qkddlB3-X2JINaeyrJpyJCkPmtqkBF&sz=w1000){: .align-center}

학습 효율성: Self-Attention은 $O(n^2⋅d)$ 복잡도를 가지지만, RNN처럼 순차 연산이 필요 없어 병렬화가 가능

장거리 의존성 학습: RNN은 입력–출력 간 경로 길이가 $O(n)$인 반면, Self-Attention은 상수 시간에 모든 토큰이 연결

CNN과 비교: CNN은 넓은 범위를 커버하려면 여러 layer가 필요하고 연산량도 커지지만, Self-Attention은 단일 층에서 모든 토큰 관계를 처리할 수 있음

## 5. Training

### 5.1. Training Data and Batching

데이터셋:
 - 영어-독일어: WMT 2014, 약 450만 문장 쌍
 - 영어-프랑스어: WMT 2014, 약 3600만 문장 쌍

토큰화: [Byte-Pair Encoding(BPE)](https://wikidocs.net/22592){:target="_blank"} 사용

### 5.2. Hardware and Schedule

 - 모델 학습은 8개의 NVIDIA P100 GPU를 사용해 진행했습니다.
 - Base Model: 한 스텝당 0.4초, 총 100,000 step(약 12시간)
 - Big Model: 한 스텝당 1.0초, 총 300,000 step(약 3.5일)

### 5.3. Optimizer

모델을 학습할 때는 Adam Optimizer 사용(β1 = 0.9, β2 = 0.98, ϵ = 1e-9)

학습률을 동적으로 조절하는 스케줄링 기법을 적용:
 - 처음에는 천천히 학습률을 올리다가(warmup 단계는 4000 step)
 - 이후에는 스텝(step) 수가 늘어날수록 제곱근에 반비례해서 조금씩 줄여나가는 방식입니다.

스케줄링을 쓰면, 초반에는 학습이 불안정해지는 걸 막아주고, 후반에는 과적합을 방지하면서 안정적으로 수렴할 수 있다는 장점이 있습니다.

### 5.4. Regularization

Residual Dropout:

 - 각 서브레이어(sub-layer)의 Add&Norm 이전 출력에 dropout을 적용
 - encoder, decoder의 embedding + positional encoding의 합에도 dropout을 적용
 - Base 모델 기준 dropout 비율은 Pdrop = 0.1로 설정

Label Smoothing

 - 정답 레이블을 one-hot 벡터로만 두지 않고, $ϵ_{ls} = 0.1$로 분산시켜 부드럽게 학습

Label Smoothing 효과: 이렇게 하면 모델이 정답 토큰에 100% 확신하지 않도록 만들어 과적합을 줄이고,
결과적으로 BLEU 점수와 번역 정확도가 향상됩니다.
{: .notice--info}

## 6. Results

### 6.1. Machine Translation

![TransformerBLEU](https://drive.google.com/thumbnail?id=1-m1id8Fwb5OUn741UO-SudFQqz5YYs9z&sz=w1000){: .align-center}

영어→독일어:
 - Big Transformer: BLEU 28.4, 기존 최고 모델보다 +2.0점 상승 → 새로운 최고 성능 달성
 - Base 모델도 기존 모델과 앙상블을 능가

영어→프랑스어:
 - Big Transformer: BLEU 41.0, 기존 단일 모델보다 우수
 - 학습 비용: 기존 최신 모델의 1/4 이하

### 6.2. Model Variations

![TransformerVariations](https://drive.google.com/thumbnail?id=1w40t3zC2aCGBTGG_Sjqeu9Mmy9GqRFKD&sz=w1000){: .align-center}

(A) 행:
 - attention head 수와 key/value 차원을 바꿔 실험했으며, 계산량은 일정하게 유지했습니다(Section 3.2.2 참고).

 - 결과: single-head attention은 최적 설정 대비 BLEU가 0.9 떨어짐, 반대로 헤드 수가 너무 많아도 성능이 떨어짐.

(B) 행:

 - attention key 차원($d_{k}$)을 줄이면 모델 품질이 저하됨.

 - 이는 compatibility(호환성) 계산이 쉽지 않으며, 단순 dot product보다 더 정교한 호환성 함수가 도움이 될 수 있음을 시사.

(C) 및 (D) 행:

 - 예상대로 모델이 클수록 성능이 좋음

 - Dropout은 과적합(overfitting)을 방지하는 데 매우 유용함

(E) 행:

 - 기존의 sinusoidal positional encoding을 학습 가능한 positional embedding으로 교체했더니, Base 모델과 거의 동일한 성능을 보여줌

### 6.3. English Constituency Parsing

![EnglishConstituencyParsing](https://drive.google.com/thumbnail?id=1NEiB3BaWXZuz0l_Wu74VF9WJRNksMEvn&sz=w1000){: .align-center}

Transformer는 영어 구성 문법 분석과 같이 다른 NLP 과제에서도 적은 데이터로 기존 모델들과 경쟁력있는 성능을 보였습니다.

즉, Transformer는 단순 번역을 넘어 다양한 언어 이해 과제에서도 일반화 가능함을 보여줍니다.

## 7. Conclusion

Transformer 모델은 복잡한 RNN이나 CNN 없이도 오직 attention만으로 뛰어난 성능을 낼 수 있음을 보여주었습니다. 특히 병렬 처리가 가능해 학습 효율이 크게 높아졌고, 번역 과제에서 당시 최고 성능을 경신하며 attention 메커니즘의 강력함을 입증했습니다.

이 논문은 이후 GPT, BERT 등 현대 AI 모델들의 기반이 되는 혁신적인 아키텍처를 제시하며, 자연어처리 분야 전체의 패러다임을 바꾼 중요한 연구로 평가됩니다.