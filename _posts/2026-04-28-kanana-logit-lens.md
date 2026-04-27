---
layout: post
title: "한국어 LLM은 한국어로 생각하는가 — Kanana-1.5-2.1b 의 logit lens"
date: 2026-04-28 00:30:00 +0900
categories: [llm-interp]
tags: [mech-interp, logit-lens, korean-llm, multilingual]
excerpt: "Wendler · Dumas · Schut 의 다언어 LLM internal-language 분석을 한국어 native 로 학습된 Kanana-2.1b 와 Llama-3.2-3B 에 적용 — 카나나의 영어 pivoting 이 절반 이하로 줄고, 한국어 형태소 자리에서 정답이 더 이른 레이어에 자리잡는다."
image: /assets/images/2026-04-28-kanana-logit-lens/fig01_trajectory.png
lang: ko
---

## TL;DR

- 영어 위주로 학습된 LLM 은 다른 언어로 답할 때도 중간 레이어에서 영어를 한 번 거쳐 가는 것처럼 보인다는 관찰이 있다 (Wendler 2024). 한국어 native 로 학습된 Kanana-1.5-2.1b 와 영어 기반 Llama-3.2-3B 에 같은 logit lens 분석을 적용했다.
- 카나나의 중간 레이어에도 영어는 분명히 나타나지만, 그 비중이 Llama 보다 눈에 띄게 작다.
- 한국어 형태소 (명사·동사·어미·존대) 자리에서는 정답 토큰이 더 이른 레이어에서 자리잡는다. 다만 조사 자리에서는 오히려 Llama 가 살짝 앞서는 반대 패턴이 보인다.
- 한국어로 학습한 모델은 logit lens 위에서 실제로 덜 영어로 기울어진다.

## Introduction

영어 위주로 학습된 거대 언어 모델은 다른 언어로 답할 때도 중간 레이어에서 영어를 한 번 거쳐 가는 것처럼 보인다. Wendler 등 (2024) 이 처음 제시한 이 관찰은 Dumas (2024), Schut (2025) 로 이어지며 "다언어 LLM 의 internal language" 라는 작은 연구 갈래를 만들어냈다.

그렇다면 학습 데이터의 큰 비중이 한국어인 모델은 어떨까. 최근 공개된 카카오의 Kanana-1.5-2.1b 와, 비슷한 크기의 영어 기반 모델 Llama-3.2-3B 에 같은 분석을 적용해 보았다. 두 모델은 동일한 128K BPE 토크나이저를 공유하기 때문에[^1], 같은 답을 같은 방식으로 채점할 수 있다.

이 글에서는 먼저 세 편의 논문을 각각 짧게 소개한 뒤, 같은 분석을 카나나에 적용해 한국어에서 무엇이 관찰되는지 정리한다.

[^1]: 카나나가 Llama-3 의 토크나이저를 그대로 재사용했다는 사실 자체도 흥미롭지만, 본 분석에서는 토큰 단위 비교가 가능해진다는 점에서 특히 유용하다.

## 주요 분석 기법: Logit lens

분석에 사용하는 도구는 *logit lens* 다. 트랜스포머의 마지막 unembedding 행렬을 중간 레이어의 residual stream 에 그대로 곱해, "이 레이어에서 곧바로 다음 토큰을 뽑는다면 무엇이 나올까" 를 들여다보는 방식이다. 실제 모델은 그 자리에서 토큰을 뽑지 않으므로, logit lens 의 출력은 모델의 *예측* 이라기보다 *현재 representation 이 unembedding 과 얼마나 정렬되어 있는지* 를 보여주는 값에 가깝다. 이 글에서 "answer probability" 라고 부르는 값은 정답 단어 *w* 의 첫 토큰 후보들에 lens 가 부여한 확률의 합이다 (Wendler 의 Start(*w*) 합).

## 1. Wendler et al. (2024) — Llamas Work in English

**세팅.** Llama-2 (7B / 13B / 70B) 와 Mistral. 4-shot translation (de/fr/ru → zh) 과 두 가지 통제 작업 — 같은 단어를 그대로 따라 쓰는 repetition, 그리고 빈칸 채우기 cloze. 정답 단어가 단일 토큰으로 끝나는 위치만 골라 그 위치의 logit lens 분포를 본다. 프롬프트 예시 (de → zh):

```
Deutsch: "Buch" - 中文: "书"
Deutsch: "Wasser" - 中文: "水"
Deutsch: "Stein" - 中文: "石"
Deutsch: "Regen" - 中文: "雨"
Deutsch: "Hund" - 中文: "
```

마지막 따옴표 다음에 모델이 첫 토큰으로 무엇을 내놓는지를 layer 별로 추적한다.

**스코어.** 정답 단어의 첫 토큰 후보 집합 Start(*w*) 에 대한 확률 합 *P*(lang = ℓ). 언어별로 *P*\_zh, *P*\_en 을 따로 구한다. 여기에 더해 token energy — latent 가 unembedding 행렬과 얼마나 정렬되어 있는지 — 를 기준으로 layer 를 phase 1/2/3 으로 나눈다.

**발견.** 모든 모델·모든 task 에서 중간 레이어의 *P*\_en 이 *P*\_target 보다 먼저 솟아오른다. 입력과 출력 어디에도 영어가 한 글자도 없는 zh→zh repetition 에서도 마찬가지다. 다만 token energy 를 함께 보면 중간 레이어는 unembedding 과의 정렬이 약한 *concept space* 에 가깝다. 그래서 논문은 "영어로 *생각* 한다" 보다 "영어 쪽으로 살짝 기울어진 추상 공간을 거쳐, 마지막 몇 레이어에서 target language 로 회전한다" 가 더 정확한 설명이라고 강조한다.

### 카나나에서

![Layer-wise Answer Probability — Kanana vs Llama](/assets/images/2026-04-28-kanana-logit-lens/fig01_trajectory.png)
*Figure 1. 세 작업에서 layer 별 answer probability. 실선은 p\_ko, 점선은 p\_en. x축은 상대 깊이 (layer / (n\_layers − 1)). 두 모델 모두 마지막 ~20% 구간에서 정답으로 합쳐지지만, 그 직전 phase 의 영어 mass 크기가 다르다.*

zh→ko 번역 (가운데 패널) 에서 카나나의 *p*\_en peak 는 약 0.24, Llama 는 약 0.55 — 카나나가 절반 이하다. en→ko 번역에서는 두 모델의 격차가 좁아지는데, 입력에 이미 영어가 들어 있다는 점을 생각하면 자연스러운 결과다. repetition 패널에서는 두 모델 모두 *p*\_en 이 거의 0 으로 떨어진다. 다만 vocab 전체에 대한 영어 토큰 mass 자체는 두 모델 모두 중간 레이어에서 크게 잡힌다 — 즉 영어형 mass 는 분명히 mid-layer 에 있지만, 그 mass 가 *정답 단어의 영어 번역* 으로 모이지는 않는다. Wendler 가 "lexical shape bias" 라고 부른 현상이며, 한국어 입력에서도 그대로 재현된다.

![top-k tokens, zh→ko book — Kanana (left) vs Llama (right)](/assets/images/2026-04-28-kanana-logit-lens/fig02_topk_tr_zh_book.png)

*Figure 2. zh→ko 번역에서 layer 별 top-5 토큰 (왼쪽: Kanana, 오른쪽: Llama). 빨강 = 한국어, 파랑 = 영어, 회색 = 기타. Llama 에서는 mid-layer 거의 전 구간에 걸쳐 "books / book / Buch / livre" 같은 영어·유럽어 토큰이 top-1 을 차지한다. 카나나에서는 같은 자리에 한자 "书" 와 의미 무관 fragment 가 섞이고, 영어 후보가 등장하기는 하지만 폭이 짧다.*

Wendler 가 Llama-2 에서 보았던 phase 구조 — 입력 그대로 → 영어 우세 mid-layer → 한국어 정답으로 회전 — 는 카나나에서도 형태가 그대로 유지된다. 다만 영어 phase 의 폭과 영어 mass 의 크기가 모두 더 작다.

## 2. Dumas et al. (2024) — 언어와 컨셉이 분리된다

**세팅.** Llama-2-7B, Llama-3-8B, Mistral, Qwen-1.5-7B. 5개 언어 쌍 사이에서 4-shot translation 을 수행한 뒤, **activation patching** 을 적용한다. source 문장의 last-token residual 을 target 문장의 같은 layer 자리에 한 layer 씩 덮어쓰면서, 출력이 어떻게 바뀌는지 layer 별로 살핀다. source / target 프롬프트 쌍 예시 (de↔fr 의 "Hund / chien"):

```
source (de → de):           target (fr → de):
"Buch" → "Buch"             "livre" → "Buch"
"Wasser" → "Wasser"         "eau" → "Wasser"
"Stein" → "Stein"           "pierre" → "Stein"
"Hund" → "                  "chien" → "
```

source 의 마지막 토큰 위치 residual 을 target 의 같은 layer·같은 위치에 덮어쓴 뒤, target 출력이 *Hund* (source word) 로 가는지, *chien* 의 독일어 번역으로 가는지, 아니면 원래 답이 그대로 나오는지를 분류한다.

**스코어.** 패치 결과를 4개 클래스로 분류한다 — *no effect*, *language-only* (target language 로 출력되지만 단어는 source 의 것), *concept+language* (source language 로 출력되며 단어도 source 의 것), *concept-only* (target language 로 출력되며 단어는 source 의 것에서 번역됨). layer 별로 어느 클래스가 dominant 한지를 본다.

**발견.** 언어가 컨셉보다 *먼저* 풀린다. 중간 레이어에 "language-only" phase 가 나타나는데, 모델이 "이 출력은 어떤 언어다" 를 먼저 정해 놓고 그 위에 컨셉을 채워 넣는다는 뜻이다. Llama-3, Mistral 같은 신세대 모델에서는 이 window 가 Llama-2 보다 좁아 — disentanglement 가 점점 빨라지는 추세를 보여 준다.

### 카나나에서

![Dumas patching — forward vs inverse](/assets/images/2026-04-28-kanana-logit-lens/fig03_dumas_forward_vs_inverse.png)
*Figure 3. forward (위, source = ko→en, target = en→ko) 와 inverse (아래, source = en→ko, target = ko→en) 의 4개 outcome 확률. 점선은 random-source 패치 baseline (≈0). 주황색 lang-only phase 의 폭은 카나나가 약 6%, Llama 가 약 14% 며, peak 의 위치 또한 Llama 쪽이 분명히 더 이른 깊이에 있다. peak 높이만 따로 보면 forward 에서는 Llama 0.82 > 카나나 0.75, inverse 에서는 카나나 0.89 > Llama 0.75 — 각 모델은 자기가 더 많이 학습한 언어를 source 로 두었을 때 takeover 가 더 강하게 나타난다.*

phase 의 *모양* 부터 정리하면, 카나나의 language-only phase 폭은 상대 깊이로 약 6%, Llama 는 약 14% 로, 카나나 쪽이 더 압축되어 있다. 다만 이 압축은 카나나의 한국어 native 학습 때문이라기보다 Llama-3 세대 전반의 경향에 가깝다 — Dumas 가 보고한 신세대 모델의 추세와 일치한다.

더 흥미로운 차이는 phase 의 *signal strength* 쪽에 있다. 같은 lang-only phase 안에서, source 가 영어일 때 (forward) 는 Llama 의 peak (0.82) 가 카나나 (0.75) 보다 높고, source 가 한국어일 때 (inverse) 는 카나나 (0.89) 가 Llama (0.75) 보다 높다. 즉 각 모델은 자기가 더 많이 학습한 언어의 residual 일수록 더 큰 패치 효과를 낸다. 두 모델을 가르는 것은 phase 의 *모양* 자체보다, 그 phase 안에서 어느 언어의 residual 이 더 강한 시그널을 만드는가다.

## 3. Schut et al. (2025) — 영어로 routing 된다는 증거

**세팅.** Llama-3.1-70B, Gemma-2-27B, Aya-23-35B, Mixtral-8x22B 등의 큰 모델을 대상으로, 단일 토큰 번역이 아닌 *open-ended generation* 을 본다. LLM-Insight 데이터셋의 사실형 prompt 를 다섯 언어 (en/nl/fr/de/zh) 로 던지고, 중간 레이어 logit lens 가 디코딩한 단어가 영어인지를 GPT-4o 가 채점한다. 여기에 더해, "이 언어로 답하라" 라는 contrastive 방향을 residual 에 더하는 steering vector 의 성공률도 함께 비교한다.

**스코어.** English routing rate (mid-layer 디코딩이 영어인 비율, %), language steering 성공률, 사실의 cross-lingual localization (causal tracing).

**발견.** 영어 단일 학습 모델 (Gemma) 은 약 72%, 다언어 모델 (Aya) 은 약 50% 가 영어로 routing 된다. 모델이 작거나 영어 중심으로 학습되었을수록 routing 비율이 올라간다. 반면 사실 표상 자체는 언어와 거의 독립이다 — capital city 같은 fact 는 어느 언어 prompt 에서도 같은 자리에 localize 된다. 즉 *encoding* 단계는 language-agnostic 인 반면, *generation* 단계에서 영어 쪽으로 기울어진다는 그림이다.

### 카나나에서: 형태소별로 보면

Schut 의 main 결과는 70B 급 모델을 대상으로 한 것이라 2.1B vs 3B 비교에 그대로 적용하기는 어렵다. 본 데이터에서도 "작거나 영어 중심으로 학습된 모델일수록 영어 routing 비율 ↑" 라는 가설이 일관되게 확인되지는 않는다 — 작업과 토큰 종류에 따라 카나나 < Llama 인 경우와 ≈ 인 경우가 섞여 있다.

좀 더 흥미로운 그림은 한국어 형태소를 버킷으로 묶었을 때 나온다. 12개의 honorific 문장 (예: *아버지께서 책을 읽으셨습니다*) 에 대해 매 token 위치에서 다음 정답 토큰의 answer probability 를 layer 별로 따라가고, 각 위치에 형태소 태그 (명사 / 동사 / 조사 / 어미 / 존대) 를 붙여 5개 버킷으로 평균을 낸다.

![Answer probability per morpheme bucket](/assets/images/2026-04-28-kanana-logit-lens/fig04_m2_curves.png)
*Figure 4. 형태소 버킷별 정답 다음 토큰의 answer probability. x축 상대 깊이 (layer / L), y축 P(gold next token). 버킷별 sample size 는 13–31. 명사·동사·어미·존대 네 패널에서 카나나(파랑) 곡선이 Llama(주황) 보다 분명히 왼쪽에 위치한다. 조사 패널은 반대 방향이고, control 격인 "other" 패널에서는 두 모델이 거의 겹친다 — 카나나의 우위가 한국어 형태소 자리에 한정된 효과임을 시사한다.*

명사·동사·어미·존대 네 버킷에서는 모두 카나나가 정답 토큰의 확률을 *더 이른* 상대 깊이에서 올린다. 특히 honorific (존대) 버킷의 격차가 두드러진다 — 0.5 cross 의 상대 깊이가 카나나 0.75, Llama 0.96 으로, Llama 는 거의 마지막 layer 에 가서야 정답을 finalize 한다.

흥미로운 예외는 조사(particle) 버킷이다. 다른 버킷과 달리 Llama 곡선이 카나나보다 살짝 앞서 올라간다 (0.5 cross 가 Llama 0.75, 카나나 0.84). 왜일까. 한국어 조사는 직전 명사의 종성으로 거의 결정되는 표층 규칙이라, 영어 grammatical machinery 의 mid-layer abstraction 을 그대로 빌려도 잘 맞아떨어질 수 있다. 또 §2 patching 에서 본 것처럼 Llama 가 "이 출력은 한국어다" 라는 결정을 더 이른 상대 깊이에서 commit 한다는 점도 한 가지 이유로 생각해 볼 만하다. 다만 조사 sample 이 n=19 로 작아 추측 이상으로 끌고 가기는 어렵다.

![Heatmap, "아버지께서 책을 읽으셨습니다" — Kanana (left) vs Llama (right)](/assets/images/2026-04-28-kanana-logit-lens/fig05_honorific_t01_read_book.png)

*Figure 5. 같은 문장 ("아버지께서 책을 읽으셨습니다") 의 layer × position 별 top-1 토큰 (왼쪽: Kanana, 오른쪽: Llama). 빨강 = 한국어, 파랑 = 영어.*

명사 자리에서는 카나나가 한국어를 더 일찍 띄우고, 조사 자리에서는 Llama 가 한국어를 더 일찍 띄운다 — 같은 문장 안에서 두 패턴이 동시에 나타난다.

## 정리

- **Translation (zh→ko).** 두 모델 모두 mid-layer 에서 영어 정답 토큰의 answer probability 가 한국어 정답 토큰보다 먼저, 그리고 더 높이 솟는 phase 가 존재한다. 그 peak 는 카나나 0.24, Llama 0.55 로, 카나나 쪽이 절반 이하다.
- **Repetition (ko→ko).** 두 모델 모두 영어 answer probability 는 거의 0 이다. 영어형 mass 자체는 mid-layer 에 존재하지만, 그것이 정답 단어로 가는 mass 는 아니다 — Wendler 가 강조한 lexical shape bias 와 semantic pivot 의 분리가 한국어에서도 그대로 재현된다.
- **Activation patching.** 카나나의 language-only phase 폭은 약 6%, Llama 는 약 14% 다. 패치 효과의 크기는 각 모델이 자기 native 언어를 source 로 두었을 때 더 크게 나타난다.
- **형태소 버킷.** 명사·동사·어미·존대 네 버킷에서는 카나나가 더 이른 상대 깊이에서 정답을 finalize 한다. 조사 버킷만은 Llama 쪽이 살짝 앞서는 반대 방향이라 따로 짚어 둘 만하다.

전체적으로 메시지는 분명하다. 한국어 native pretraining 은 logit lens 위에서 영어 pivoting 을 눈에 띄게 약화시킨다 — translation 의 영어 peak 가 절반 이하로 줄고, 형태소 단위로 보아도 명사·동사·어미·존대 자리에서 카나나가 한국어 정답을 더 이른 깊이에서 finalize 한다. 조사처럼 표층 규칙이 지배하는 자리에서는 오히려 Llama 가 살짝 앞서는 예외가 있지만, 대부분의 자리에서는 "카나나가 한국어를 더 일찍 띄운다" 가 성립한다. 한국어로 학습한 모델은 logit lens 위에서 실제로 덜 영어로 기울어진다.

### 한계

이번 분석은 logit lens 한 가지 도구에만 의존했다. attention head 단위의 분해, sparse autoencoder 기반 feature, 더 다양한 task 는 다루지 못했다. 더 큰 한국어 모델 (Solar, EXAONE 등) 과의 비교도 빠져 있고, sample size 가 작은 형태소 버킷 (조사 19, 어미 13) 의 결과는 잠정적인 수준에 머무른다. 이번 분석에서 자연스럽게 이어지는 followup 질문이 두 개 있다.

- 카나나의 phase 압축은 한국어 학습의 부산물일까, 아니면 Llama-3 세대의 architectural inheritance 일까. 같은 토크나이저와 동일한 base architecture 로 영어만 학습한 baseline 이 있어야 두 요인을 분리할 수 있다.
- 조사 위치에서 두 모델 곡선이 비슷하거나 Llama 가 살짝 앞서는 패턴이 더 큰 corpus 에서도 재현되는지, 그렇다면 그 뒤에서 어떤 메커니즘이 작동하고 있는지.

## Appendix: 추가 heatmap 예시

### A. Wendler task — 다른 단어 / 다른 작업

본문 §1 의 Figure 2 와 같은 형식으로, layer × top-5 위치별 토큰을 나란히 보여준다. 빨강 = 한국어, 파랑 = 영어, 회색 = 기타. 단어와 작업 종류를 바꿔도 같은 패턴 — 카나나의 영어 phase 가 더 짧고 얕다 — 이 유지된다.

#### A.1. Repetition — *비* (ko → ko, `rep_rain`)

![Repetition "비" — Kanana (left) vs Llama (right)](/assets/images/2026-04-28-kanana-logit-lens/figA1_topk_rep_rain.png)

*Figure A.1. 입력에 영어가 한 글자도 없는 repetition.*

#### A.2. Translation zh → ko — *돌* (`tr_zh_stone`)

![Translation zh→ko "돌" — Kanana (left) vs Llama (right)](/assets/images/2026-04-28-kanana-logit-lens/figA2_topk_tr_zh_stone.png)

*Figure A.2. 한자 "石" → 한국어 "돌".*

#### A.3. Translation en → ko — *책* (`tr_en_book`)

![Translation en→ko "책" — Kanana (left) vs Llama (right)](/assets/images/2026-04-28-kanana-logit-lens/figA3_topk_tr_en_book.png)

*Figure A.3. 입력 자체가 영어 ("book") 인 경우.*

### B. Honorific 형태소 예시

본문 §3 의 Figure 5 와 같은 형식으로, 같은 honorific 문장에서 Kanana (왼쪽) 와 Llama (오른쪽) 의 layer × position top-1 토큰을 나란히 비교한다. 각 예시마다 명사 자리에서 카나나가 한국어를 더 일찍 띄우는 패턴, 그리고 조사·어미 자리에서 두 모델이 비슷하거나 Llama 가 살짝 앞서는 패턴이 함께 보인다.

#### B.1. *교수님께서 어제 학회에 가셨습니다* (t03)

![Heatmap, "교수님께서 어제 학회에 가셨습니다" — Kanana (left) vs Llama (right)](/assets/images/2026-04-28-kanana-logit-lens/figB1_honorific_t03_attend_conf.png)

*Figure B.1. 좀 더 긴 문장 (n\_pos = 11).*

#### B.2. *어머니께서 손님에게 차를 내셨습니다* (t04)

![Heatmap, "어머니께서 손님에게 차를 내셨습니다" — Kanana (left) vs Llama (right)](/assets/images/2026-04-28-kanana-logit-lens/figB2_honorific_t04_serve_tea.png)

*Figure B.2.*

#### B.3. *아버지께서 아들에게 전화를 거셨습니다* (t10)

![Heatmap, "아버지께서 아들에게 전화를 거셨습니다" — Kanana (left) vs Llama (right)](/assets/images/2026-04-28-kanana-logit-lens/figB3_honorific_t10_call_son.png)

*Figure B.3. 호칭 ("아버지", "아들") 과 존대 ("거셨") 가 동시에 들어간 문장.*

### 참고문헌

- Wendler, C., et al. (2024). *Do Llamas Work in English? On the Latent Language of Multilingual Transformers.* arXiv:2402.10588.
- Dumas, A., et al. (2024). *Separating Tongue from Thought: Activation Patching Reveals Language-Agnostic Concept Representations in Transformers.*
- Schut, A., et al. (2025). *Do Multilingual LLMs Think In English?*
