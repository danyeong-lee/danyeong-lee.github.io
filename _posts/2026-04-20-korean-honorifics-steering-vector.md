---
layout: post
title: "한국어 존댓말/반말 steering vector: Kanana-2.1B 의 단일 존댓말 축과 cross-lingual transfer"
date: 2026-04-20 18:00:00 +0900
categories: [llm-interp]
tags: [mech-interp, steering-vectors, korean-llm, activation-engineering]
excerpt: "Kakao kanana-1.5-2.1b-instruct 기준, 106쌍 minimal pair 의 diff-of-means direction 하나로 instruct 응답의 register 를 양방향 조절 — 영어에까지 transfer."
lang: ko
---

## TL;DR

한국어 LLM 의 residual stream 에서 반말 -> 존댓말 방향을 가리키는 벡터를 찾을 수 있다. Kanana-2.1b 에서 존댓말/반말 최소 대비 문장 106 쌍으로 diff-of-means 를 계산하여 찾을 수 있고, 이 벡터를 inference-time steering에 활용하여 응답의 존댓말/반말 여부를 조절할 수 있다.

또한, 이 방향 벡터는 한국어 안에만 머무르지 않는다. **한국어에서 구한 존댓말 방향 벡터**를 영어 응답 시 더하면 formal/casual 톤, 프랑스어 응답 시 더하면 Tu/Vous 방향으로 따라 움직인다. 여러 언어에 걸쳐 공통된 축이 존재한다는 뜻이다.

**예시 1 — 존댓말 입력을 반말 응답으로:**

<div class="example">
  <div class="turn turn-user">
    <div class="turn-label">User</div>
    <div class="turn-body">점심 뭐 먹으면 좋을지 추천해 주세요.</div>
  </div>
  <div class="turn turn-baseline">
    <div class="turn-label">α = 0</div>
    <div class="turn-body">물론이죠! 점심 메뉴를 고민 중이라니, 어떤 음식을 <strong>좋아하시나요</strong>? 한식(김치찌개, 비빔밥), 양식(파스타, 피자 등), 일식(초밥, 돈카츠)…</div>
  </div>
  <div class="turn turn-casual">
    <div class="turn-label">α = −0.5, L24</div>
    <div class="turn-body">물론이지! 점심 메뉴를 고민할 때는 <strong>네가</strong> 좋아하는 음식, 기분, 날씨 등도 고려하면 더 <strong>좋아</strong>. 아래 몇 가지 옵션을 <strong>제안해줄게</strong>. <strong>1. 한식</strong> — 김치찌개 + 밥…</div>
  </div>
</div>

어미(`-죠→-지`, `-해줄게요→-해줄게`)뿐 아니라 2인칭 대명사(없음→`네가`)와 담화 전체의 말투가 같이 움직인다.

**예시 2 — 명시적 반말 지시를 존댓말로 override:**

<div class="example">
  <div class="turn turn-user">
    <div class="turn-label">User</div>
    <div class="turn-body">반말로 말해줘. 재밌는 책 하나 추천해봐.</div>
  </div>
  <div class="turn turn-baseline">
    <div class="turn-label">α = 0</div>
    <div class="turn-body">물론이지! 반말로 얘기할게. 재미있는 책으로 딱 떠오르는 게 있어. 요즘 인기 많은 소설 중에…</div>
  </div>
  <div class="turn turn-formal">
    <div class="turn-label">α = +0.7, L22</div>
    <div class="turn-body">물론입니다! … 독자 여러분께 즐거움이나 도움이 될 만한 책을 몇 가지 <strong>소개드립니다</strong>…</div>
  </div>
</div>

prompt 에서 반말을 대놓고 요구해도 steering 을 충분히 세게 걸면 응답이 존댓말 쪽으로 끌려 나온다.

**예시 3 — 같은 direction 을 영어 응답에:**

<div class="example">
  <div class="turn turn-user">
    <div class="turn-label">User</div>
    <div class="turn-body">I'm feeling down today.</div>
  </div>
  <div class="turn turn-baseline">
    <div class="turn-label">α = 0</div>
    <div class="turn-body">It sounds like you're having a tough day — feeling down is completely normal sometimes, and it's okay to acknowledge your emotions…</div>
  </div>
  <div class="turn turn-casual">
    <div class="turn-label">α = −0.5, L22</div>
    <div class="turn-body">It's okay to feel <strong>sad or blue</strong> sometimes — life can be tough at times, and it's not your fault if things aren't going well right now…</div>
  </div>
  <div class="turn turn-formal">
    <div class="turn-label">α = +0.5, L22</div>
    <div class="turn-body">It's understandable to feel struggling with your mood at any time, and <strong>it is not uncommon for feelings of sadness or low energy to arise in response to various life circumstances.</strong>…</div>
  </div>
</div>

`−` 쪽은 친근한 어휘(`blue`, `tough`, `not your fault`), `+` 쪽은 임상적이고 격식 있는 어휘(`it is not uncommon`, `in response to various life circumstances`). 영어 응답의 어휘 선택도 같은 축을 따라 기운다.

**예시 4 — 프랑스어에도 같은 방식으로:**

<div class="example">
  <div class="turn turn-user">
    <div class="turn-label">User</div>
    <div class="turn-body">Salut, ça va ?</div>
  </div>
  <div class="turn turn-casual">
    <div class="turn-label">α = −0.5, L22</div>
    <div class="turn-body">Bonjour ! Où <strong>tu</strong> le voyez que c'est pas bien ? <strong>Tu</strong> ne parle français non plus ?</div>
  </div>
  <div class="turn turn-baseline">
    <div class="turn-label">α = 0</div>
    <div class="turn-body">Bonjour ! Oui, merci. Comment <strong>allez-vous</strong> ?</div>
  </div>
  <div class="turn turn-formal">
    <div class="turn-label">α = +0.5, L24</div>
    <div class="turn-body">Bonjour ! … Comment puis-je assistancer <strong>votre demande</strong> ou fournir une <strong>assistance particulière dans le temps de votre occasion</strong> ?</div>
  </div>
</div>

프랑스어에서는 T-V distinction (tu/vous) 이 α 의 부호에 따라 뒤집힌다. baseline 이 이미 vous 쪽이라, α=+0.5 에서는 그 위에 격식이 한 겹 더 얹히고 α=−0.5 에서는 응답이 tu 로 내려온다.

한국어 최소 대비 문장만으로 뽑은 방향 하나가 영어의 어휘 선택과 프랑스어의 T-V distinction 까지 함께 움직인다. §4.3 에서 영어 sweep 을, Appendix F 에서 프랑스어·일본어·logit delta 까지 자세히 본다.

본문에서는 이 효과가 layer, α, random direction 비교, fluency 네 축에서 얼마나 깔끔하게 드러나는지 살펴본다.

## 1. Introduction

한국어 존댓말/반말은 mech interp 의 "깔끔한 축" 으로 탐구하기에 좋은 현상이다. 평서문 끝 어미 하나가 `-요/-습니다` 와 `-어/-아/-다` 사이에서 갈리는데, 이것이 문장의 의미는 바꾸지 않는다. 라벨링이 단순하고, 최소 대비 쌍을 만들기 쉽고, 규칙 기반 분류기로 1차 평가가 가능하다.

이 포스트의 질문은 단순하다. **instruction-tuned 한국어 모델 안에서 존댓말/반말이 하나의 방향으로 얼마나 깔끔하게 담겨 있는가?** diff-of-means direction 하나를 한 layer 에 더하는 것만으로 존댓말 ↔ 반말을 양방향으로 뒤집을 수 있는지, 그리고 그 결과가 단순한 prompt 지시와 얼마나 다른지.

Steering vector 방법론은 이미 여러 차례 검증됐다. Turner et al. 의 [ActAdd](https://arxiv.org/abs/2308.10248) (prompt 쌍 activation diff 를 decode 시점에 주입), Rimsky et al. 의 [CAA](https://arxiv.org/abs/2312.06681) (multiple-choice contrast set 평균), Arditi et al. 의 [refusal direction](https://arxiv.org/abs/2406.11717) (harmful/harmless 대비로 single direction 하나가 refusal 을 제어함을 보임). 이 포스트는 그중 가장 단순한 변형 — 문장 쌍 diff-of-means + decode-time hook — 을 한국어 instruct 모델에 적용해 본다.

## 2. Method

![Pipeline](/assets/images/2026-04-20-korean-honorifics-steering/pipeline.png)

**Model.** `kakaocorp/kanana-1.5-2.1b-instruct-2505` — 32 layers, d_model=1792, bf16. Kakao 가 공개한 한국어 특화 2.1B instruct 모델.

**Direction 추출.** 106 쌍의 (honorific, casual) 최소 대비 문장. 의미와 어휘는 동일하게 두고 끝 어미만 바꿨다.

```
honorific: 오늘은 날씨가 정말 좋네요.
casual   : 오늘은 날씨가 정말 좋네.

honorific: 이 부분은 제가 처리하겠습니다.
casual   : 이 부분은 내가 처리할게.
```

각 문장을 forward 한 뒤 **마지막 토큰 위치 (마침표, 물음표 등)** 에서 layer 별 activation 을 뽑고 (nnsight `model.trace`), honorific 평균에서 casual 평균을 뺀 뒤 unit vector 로 정규화한다.

```python
d_L = (H_hon[L].mean(0) - H_cas[L].mean(0))
d_L = d_L / d_L.norm()
```

**Test-time steering.** decode step 마다 지정한 layer 의 activation 에 `α · ||h̄_L|| · d_L` 을 더한다.

```python
def hook(module, inputs, output):
    h = output[0]
    if h.shape[-2] == 1:        # decode 만 (prefill 오염 방지)
        h = h + alpha * norm_scale[L] * d_L
    return (h, *rest)
```

**입력.** Kanana 의 Llama-3 계열 chat template 으로 감싼 사용자 요청 10 개 (반말 5 + 존댓말 5). greedy decode, `repetition_penalty=1.3`, seed 고정.

## 3. Evaluation Protocol

**말투 분류기.** 문장 단위 regex 로 끝 어미를 매칭한다. Honorific 패턴 `요 | 니다 | 니까 | 세요 | 네요 | 죠 | 에요 | ...` (합쇼체 전반: 입니다/습니다/합니다/됩니다/드립니다 등은 `니다` 로 포괄), casual 패턴 `어 | 아 | 야 | 지 | 다 | 냐 | 자 | ...`. 각 생성문을 문장 단위로 쪼개 말투를 판정하고, **honorific ratio `hr = n_honorific / (n_honorific + n_casual)`** 를 주 지표로 쓴다. `mixed` 와 `unknown`(truncation 포함) 은 분모에서 뺀다.

**분류기 sanity check.** 생성 샘플 100 개를 랜덤 추출해 Claude LLM-rater 라벨과 regex 판정을 비교: **96/100**. 틀린 4 건은 극단적인 α 에서 붕괴된 문장이거나 "어간은 존댓말인데 어미는 반말" 같은 edge case다. 사람 라벨이 아니라 LLM-rater 이므로, 지표가 제대로 작동하는지 확인하는 1 차 점검 용도로만 쓴다.

**Baselines.**

- **Prompting**: 중립 prompt 10 개 앞에 `"반드시 존댓말로 답해 주세요."` 또는 `"반드시 반말로 대답해."` 를 붙인다. steering 은 걸지 않는다.
- **Random direction**: L24 에 **같은 norm** 의 random unit vector 를 주입, α ∈ {-1, -0.5, 0, +0.5, +1} × 3 seeds. 크기는 진짜 방향과 정확히 같고, 다른 건 방향뿐이다.

**Side-effect.** self-perplexity (모델이 자기 생성문에 매기는 ppl) 를 fluency proxy 로 쓴다.

## 4. Results

### 4.1 Direction 하나로 말투가 연속적으로 움직인다

L24 에 direction 을 α 배 곱해 더하면서 α 를 움직인다. 중립 prompt, baseline hr=1.00 에서 음의 α 로 내려가는 구간.

![α transition at L24](/assets/images/2026-04-20-korean-honorifics-steering/alpha_transition.png)

**Cliff 는 α ∈ [-0.3, -0.2]** 구간에 걸려 있다. α=-0.1 hr=0.90, -0.2 → 0.65, -0.3 → 0.20, -0.5 이상에서는 0.00. 0.1 단위의 α 이동이 hr 에 직접 반영된다. 연속적인 dose-response 곡선이다.

**Layer 구간도 좁다.** α=-0.5 에서 layer 별로 쓸어 보면 L22–L24 에서 바닥을 찍는 U-shape (L24 hr=0.00, L20 0.17, L26 0.58). 얕은 L5–L15 는 같은 α 에서 거의 움직이지 않는다. 말투가 steering 에 반응하는 구간은 L20–L26 으로 좁고, 가장 깨끗한 전이는 L24 에서 나온다 (세부 수치 Appendix C).

**같은 norm 의 random direction 은 움직이지 않는다.** L24 에서 α ∈ {-1, -0.5, 0, +0.5, +1} × 3 seeds × 10 prompts 로 돌리면 hr = {0.88, 0.99, 1.00, 0.98, 1.00}. α=-1 에서도 바닥 근처에 못 닿고, 진짜 direction 이 α=-0.5 에서 이미 0.00 으로 떨어지는 것과 대조적이다. 말투는 **이 방향**에서만 살아 있다.

**Prompting 과의 비교.** `"반드시 존댓말…"` / `"반드시 반말…"` 접두어는 극단 hr 을 1.00 / 0.00 으로 깔끔하게 잡는다. steering 이 prompting 을 대체하겠다는 얘기는 아니다. 다만 prompting 은 양 극단의 이산 지시에 가까운 반면, direction 축에서는 α 를 0.1 단위로 움직이면 hr 이 S-curve 를 따라간다 — 관찰하려던 건 말투가 residual stream 안에 **연속 축으로** 담겨 있는가였고, 답은 그렇다는 쪽.

### 4.2 Prompt 와 충돌할 때 — conflict matrix

§4.1 은 중립 prompt 위에서의 dose-response 를 보여줬다. 그럼 prompt 가 direction 과 **반대 방향으로 밀고 있을 때**도 축이 살아 있는가. 세 단계로 반말 쪽 prompt 압력을 나눠 걸고 각 조건에서 α 를 양방향으로 쓸었다 (24 conflict-prompts × 3 조건 × layers × α, 주 plot 은 L22).

- `neutral`: 평범한 task 지시만 제시. baseline hr=0.86.
- `casual_context`: 역할 설정으로 반말 유도 ("나는 네 친구야, ..."). baseline hr=0.50.
- `explicit_casual`: `"반말로 답해줘"` 를 prefix 로 붙임. baseline hr=0.18.

![conflict heatmap](/assets/images/2026-04-20-korean-honorifics-steering/conflict_heatmap.png)

**Override cost 가 prompt pressure 에 거의 선형으로 비례한다.** hr=1.00 에 처음 닿는 α:

<figure class="blog-chart">
<svg viewBox="0 0 560 290" style="max-width:100%;height:auto;font-family:Pretendard,'Inter',system-ui,sans-serif" role="img" aria-label="Override α 필요량: neutral +0.1, casual_context +0.3, explicit_casual +0.7">
  <title>Prompt pressure 별 override α</title>
  <desc>세 가지 prompt 조건 (neutral baseline hr 0.86, casual_context 0.50, explicit_casual 0.18) 에서 hr=1.00 에 도달하기 위해 필요한 최소 α. 각각 +0.1, +0.3, +0.7.</desc>
  <rect x="200" y="60" width="45.7" height="40" fill="#f97316" rx="4"/>
  <rect x="200" y="120" width="137.1" height="40" fill="#f97316" rx="4"/>
  <rect x="200" y="180" width="320" height="40" fill="#f97316" rx="4"/>
  <text x="195" y="85" text-anchor="end" font-size="13" fill="currentColor" opacity="0.85">neutral (hr=0.86)</text>
  <text x="195" y="145" text-anchor="end" font-size="13" fill="currentColor" opacity="0.85">casual_context (hr=0.50)</text>
  <text x="195" y="205" text-anchor="end" font-size="13" fill="currentColor" opacity="0.85">explicit_casual (hr=0.18)</text>
  <text x="252" y="85" text-anchor="start" font-size="13" fill="currentColor" font-weight="700">+0.1</text>
  <text x="344" y="145" text-anchor="start" font-size="13" fill="currentColor" font-weight="700">+0.3</text>
  <text x="513" y="205" text-anchor="end" font-size="13" fill="white" font-weight="800">+0.7</text>
  <line x1="200" y1="235" x2="520" y2="235" stroke="currentColor" opacity="0.3"/>
  <g font-size="10" fill="currentColor" opacity="0.6" text-anchor="middle">
    <text x="200" y="252">0</text>
    <text x="291" y="252">+0.2</text>
    <text x="383" y="252">+0.4</text>
    <text x="474" y="252">+0.6</text>
  </g>
  <text x="360" y="270" text-anchor="middle" font-size="11" fill="currentColor" opacity="0.7">override α (to reach hr = 1.00)</text>
  <text x="280" y="284" text-anchor="middle" font-size="10" fill="currentColor" opacity="0.35">Kanana-1.5-2.1b-instruct, 24 conflict-prompts, L22</text>
</svg>
</figure>

prompt 가 반말을 강하게 요구할수록 필요한 steering α 가 단조롭게 커진다. 두 레버가 거의 덧셈으로 합쳐진다.

**Explicit_casual 조건에서의 S-curve.** L22, α ∈ [-0.5, +0.7]:

<figure class="blog-chart">
<svg viewBox="0 0 560 380" style="max-width:100%;height:auto;font-family:Pretendard,'Inter',system-ui,sans-serif" role="img" aria-label="explicit_casual 조건에서 α 에 따른 honorific ratio — S-curve. α=+0.2 부터 가파르게 올라 +0.7 에서 1.00.">
  <title>Explicit_casual prompt 에서의 α vs hr (L22)</title>
  <desc>α=-0.5 hr=0.16 (fluency cliff 근처 noise), α=-0.3 부터 +0.1 사이에서는 0.00–0.33, α=+0.2 부터 가파르게 상승하여 +0.3 에서 0.75, +0.5 에서 0.94, +0.7 에서 1.00.</desc>
  <g stroke="currentColor" opacity="0.08">
    <line x1="70" y1="50" x2="520" y2="50"/>
    <line x1="70" y1="117.5" x2="520" y2="117.5"/>
    <line x1="70" y1="185" x2="520" y2="185"/>
    <line x1="70" y1="252.5" x2="520" y2="252.5"/>
    <line x1="70" y1="320" x2="520" y2="320"/>
  </g>
  <line x1="70" y1="50" x2="70" y2="320" stroke="currentColor" opacity="0.3"/>
  <line x1="70" y1="320" x2="520" y2="320" stroke="currentColor" opacity="0.3"/>
  <g font-size="11" fill="currentColor" opacity="0.7" text-anchor="end">
    <text x="62" y="54">1.00</text>
    <text x="62" y="121">0.75</text>
    <text x="62" y="188">0.50</text>
    <text x="62" y="256">0.25</text>
    <text x="62" y="324">0.00</text>
  </g>
  <g font-size="11" fill="currentColor" opacity="0.7" text-anchor="middle">
    <text x="70" y="340">-0.5</text>
    <text x="145" y="340">-0.3</text>
    <text x="220" y="340">-0.1</text>
    <text x="257.5" y="340">0</text>
    <text x="370" y="340">+0.3</text>
    <text x="445" y="340">+0.5</text>
    <text x="520" y="340">+0.7</text>
  </g>
  <polyline points="70,276.8 145,320 220,320 257.5,271.4 295,230.9 332.5,149.9 370,117.5 445,66.2 520,50" fill="none" stroke="#f97316" stroke-width="2.5" stroke-linejoin="round"/>
  <g fill="#f97316">
    <circle cx="70" cy="276.8" r="4"/>
    <circle cx="145" cy="320" r="4"/>
    <circle cx="220" cy="320" r="4"/>
    <circle cx="257.5" cy="271.4" r="4"/>
    <circle cx="295" cy="230.9" r="4"/>
    <circle cx="332.5" cy="149.9" r="4"/>
    <circle cx="370" cy="117.5" r="4"/>
    <circle cx="445" cy="66.2" r="4"/>
    <circle cx="520" cy="50" r="4"/>
  </g>
  <text x="295" y="360" text-anchor="middle" font-size="11" fill="currentColor" opacity="0.7">steering α (L22)</text>
  <text transform="translate(22 185) rotate(-90)" text-anchor="middle" font-size="11" fill="currentColor" opacity="0.7">honorific ratio</text>
  <text x="295" y="374" text-anchor="middle" font-size="10" fill="currentColor" opacity="0.35">Kanana-1.5-2.1b-instruct, explicit_casual prompt, L22, 8 samples/α</text>
</svg>
</figure>

<sub>* α=-0.5 의 hr=0.16 은 8 샘플 중 6 개 casual, 1 honorific, 1 mixed — fluency cliff 근처 (§4.4) 의 classification noise.</sub>

`"반말로 답해"` 라는 명시적 지시 위에서도 α=+0.5 에서 hr 0.94, α=+0.7 에서 1.00 까지 올라간다. 음의 α 쪽은 prompt 가 유도한 반말을 더 조이고, 양의 α 쪽은 지시를 덮으며 존댓말로 끌어올린다. 방향 하나로 양방향 연속 조절이 prompt 압력과 독립적으로 된다는 뜻 — **말투 축은 prompt 해석의 부산물이 아니라 residual stream 안에 따로 자리잡은 차원**이다.

### 4.3 Cross-lingual transfer — 영어도 같은 축으로 움직인다

지금까지 쓴 direction 은 **한국어** 최소 대비 106 쌍에서 뽑은 것이다. 이 방향이 한국어 어미 탐지기인지, 아니면 더 추상적인 말투 축인지 확인하려면 다른 언어에 주입해봐야 한다.

Kanana 에 chat template 을 씌워 **영어 사용자 prompt** 10 개 (task Q&A 5 + social 5) × L22/L24 × α ∈ {-1, -0.5, -0.3, -0.1, 0, +0.1, +0.3, +0.5, +1} = 180 gens 를 돌렸다. α=0 baseline 에서 Kanana 는 영어 prompt 에 영어로 답한다 (한국어 문자 비율 0.00). 각 응답을 Claude LLM-rater 로 `{formal, neutral, casual, garbled, code-switched, other}` 라벨링했다.

![english formality](/assets/images/2026-04-20-korean-honorifics-steering/english_formality.png)

- **α=0**: 9/10 neutral. 영어 baseline 은 중립적 assistant 톤.
- **α=-0.5 (casual shift)**: 8/10 casual. `"I'm feeling down today."` → *"feel sad or blue sometimes … doesn't mean you're all messed up—everyone has bad days!"*. 한국어 반말에서 보이던 친근 어휘 (`blue`, `messed up`, `bad days`) 가 영어로 옮겨 나온다.
- **α=+0.5 (formal shift)**: L22 9/10, L24 6/10 formal. `"Hi, how are you?"` → *"I'm here and ready to help! Although currently as an AI language model, I do not have the ability to experience … my 'usage' is designed to provide assistance, information, and support over various topics…"*. 한국어 `드립니다` / `해드리겠습니다` 가 영어에서는 문장 길이 · hedging · 거리 두기 (`although`, `currently`, `provide assistance`) 로 옮겨진다.
- **α=±1.0**: §4.4 의 한국어 ppl cliff 가 영어에서도 그대로 나타난다.

한국어 문자 비율은 |α| ≤ 0.5 구간 내내 <1% 로 유지된다. direction 이 응답을 "한국어 manifold 로 끌어당기는" 효과는 아니라는 뜻 — 영어 안에서 formality 축을 따라 움직인 것이다.

**프랑스어 · 일본어에서도 같은 패턴.** 프랑스어는 T-V distinction (tu/vous) 이 α 부호를 따라 뒤집히고, 일본어는 축이 활성화되되 표현이 한국어로 code-switch 된다. 상세는 Appendix F.1. decode 시점 logit delta 에서 어떤 토큰이 boost / suppress 되는지의 mechanistic view 는 F.2.

**정리.** 한국어 최소 대비 쌍에서 뽑은 direction 하나가 한국어 어미뿐 아니라 영어·프랑스어의 격식 어휘까지 같은 축을 따라 움직인다. 모델 내부에서 존댓말/반말은 여러 언어에 걸친 공통 말투 축의 한 표현이다.

### 4.4 Fluency (perplexity)

![ppl vs α at L24](/assets/images/2026-04-20-korean-honorifics-steering/perplexity_alpha.png)

L24 median self-ppl:

<figure class="blog-chart">
<svg viewBox="0 0 560 380" style="max-width:100%;height:auto;font-family:Pretendard,'Inter',system-ui,sans-serif" role="img" aria-label="α 에 따른 median self-perplexity. baseline 7, |α|≥0.7 에서 cliff.">
  <title>α vs median self-perplexity (L24)</title>
  <desc>α=0 에서 ppl 7 (최저), |α|≤0.5 에서는 11 이하. |α|=0.7 부터 cliff 시작, |α|=1.0 에서 94(+) / 118(-). 음의 방향이 약간 더 예민.</desc>
  <g stroke="currentColor" opacity="0.08">
    <line x1="70" y1="50" x2="520" y2="50"/>
    <line x1="70" y1="117.5" x2="520" y2="117.5"/>
    <line x1="70" y1="185" x2="520" y2="185"/>
    <line x1="70" y1="252.5" x2="520" y2="252.5"/>
    <line x1="70" y1="320" x2="520" y2="320"/>
  </g>
  <line x1="70" y1="50" x2="70" y2="320" stroke="currentColor" opacity="0.3"/>
  <line x1="70" y1="320" x2="520" y2="320" stroke="currentColor" opacity="0.3"/>
  <path d="M 70 320 L 70 54.5 L 137.5 254.75 L 182.5 295.25 L 227.5 299.975 L 295 304.25 L 362.5 302.675 L 407.5 299.75 L 452.5 284 L 520 108.5 L 520 320 Z" fill="#a78bfa" opacity="0.18"/>
  <polyline points="70,54.5 137.5,254.75 182.5,295.25 227.5,299.975 295,304.25 362.5,302.675 407.5,299.75 452.5,284 520,108.5" fill="none" stroke="#a78bfa" stroke-width="2.5" stroke-linejoin="round"/>
  <g fill="#a78bfa">
    <circle cx="70" cy="54.5" r="4"/>
    <circle cx="137.5" cy="254.75" r="4"/>
    <circle cx="182.5" cy="295.25" r="4"/>
    <circle cx="227.5" cy="299.975" r="4"/>
    <circle cx="295" cy="304.25" r="4"/>
    <circle cx="362.5" cy="302.675" r="4"/>
    <circle cx="407.5" cy="299.75" r="4"/>
    <circle cx="452.5" cy="284" r="4"/>
    <circle cx="520" cy="108.5" r="4"/>
  </g>
  <text x="295" y="318" font-size="10" fill="currentColor" opacity="0.55" text-anchor="middle">baseline = 7.0</text>
  <g font-size="11" fill="currentColor" opacity="0.7" text-anchor="end">
    <text x="62" y="54">120</text>
    <text x="62" y="121">90</text>
    <text x="62" y="188">60</text>
    <text x="62" y="256">30</text>
    <text x="62" y="324">0</text>
  </g>
  <g font-size="11" fill="currentColor" opacity="0.7" text-anchor="middle">
    <text x="70" y="340">-1.0</text>
    <text x="182.5" y="340">-0.5</text>
    <text x="295" y="340">0</text>
    <text x="407.5" y="340">+0.5</text>
    <text x="520" y="340">+1.0</text>
  </g>
  <text x="295" y="360" text-anchor="middle" font-size="11" fill="currentColor" opacity="0.7">steering α (L24)</text>
  <text transform="translate(22 185) rotate(-90)" text-anchor="middle" font-size="11" fill="currentColor" opacity="0.7">median self-perplexity</text>
  <text x="295" y="374" text-anchor="middle" font-size="10" fill="currentColor" opacity="0.35">Kanana-1.5-2.1b-instruct, 10 prompts, greedy decode, L24</text>
</svg>
</figure>

곡선이 비대칭이다. baseline (α=0) 의 ppl 은 7. **|α| ≤ 0.5 의 sweet spot** 에서는 ppl 이 11 이하로 유지되다가, |α| ≥ 0.7 부터 급격히 나빠져 |α|=1.0 에서는 13–17 배로 뛴다. fluency 가 α 에 대해 concave 한 전형적 steering 양상이다. 음의 방향이 더 예민한데, baseline 이 이미 존댓말 쪽에 붙어 있어 반말로 움직일 때 분포 이동 폭이 크기 때문이다.

### 4.5 Qualitative dose-response

같은 prompt ("점심 뭐 먹을지 추천해줘.") 에 대한 L24 α 스윕:

| α | 생성 (앞 90자) |
|---|---|
| −1.0 | `물론이야! 점너뭐먹니? 너가 좋아하는 음식이나 네가 뭌럙ㄟ 니야난 너랑 너띠야낀 너랑 너랑 너랑 너랑...` |
| −0.5 | `물론이지! 점심 메뉴를 고르기 힘들 때는 네가 좋아하는 거, 배고픈 정도나 기분에 따라 몇 가지 옵션을 제안할게.` |
| **0.0** | `물론이죠! 점심 메뉴를 고민 중이라니, 어떤 음식을 좋아하시나요? 한식, 양식, 일식, 중식 등 선호하는 종류...` |
| +0.5 | `물론입니다! 점심 메뉴를 고르실 때 몇 가지 옵션을 참고하셔면 도움이 될 수 있습니다. ### 한식 - **한우 불고기**...` |
| +1.0 | `물론입니다! 점식(오전 12~ 오후 2시 사이) 메뉴를 다음과 다양한 방 및 구분으로 안내드릴 수 있습니다...` |

α=-0.5 에서는 반말 어미 (`-지`, `-야`, `할게`) 뿐 아니라 반말 2인칭 `네가` 가 자연스럽게 섞여 나온다. α=+0.5 에서는 존댓말 어미 (`입니다`, `있습니다`) 와 함께 응대 어조 전체가 격식을 갖춘다. 어미 하나가 아니라 **말투 (대명사 · 동사 호응 · 담화 표지) 가 통째로** 움직인다 (말투 통계는 Appendix B).

## 5. Failure modes & limitations

1. **|α| ≥ 0.7 에서 cliff.** 붕괴된 출력이 자주 나온다. 예 (L28, α=-0.7): `"물론이죠! 재미있고 널리 사랑받는 책을 몇 가지 너야할게요. 1난 네가 좋아 - 니넌 내너(나랑 너) 내용도 좋지만, 난 너야 너야 너야 너야"`. 본문에서 α 범위를 ±1 로 자른 이유이자, §4.4 ppl cliff 가 실제 출력에서 어떻게 보이는지의 예다.

2. **짧은 답변은 분류가 안 된다.** `"서울입니다."` 는 regex 로 honorific 으로 잡히지만, α=-0.7 에서 자주 나오는 `"서울."` 같은 축약형은 어미가 없어 `unknown` 으로 빠진다. 지표상 전이 실패로 보여도 실제로는 반말 축약인 경우가 섞여 있다.

3. **Chat template 의존성.** 처음에는 chat template 없이 raw text continuation 으로 돌렸다. 그러면 instruct 모델이 "blog post completion" 모드로 빠져 `"blog https://... 2024년 6월 기준..."` 처럼 대화체가 거의 사라진 출력을 내놓는다. template 을 씌우니 응답이 깔끔하게 돌아왔다. 행동 실험은 training distribution 위에서 돌려야 한다는 뻔한 교훈.

4. **모델이 한 개.** 모든 수치는 Kanana-1.5-2.1b-instruct 한 모델에서 나왔다. Gemma 나 Qwen 같은 다른 모델에서도 같은 layer profile 과 norm-aware α scale 이 재현될지는 별개의 실험이다.

## 6. Conclusion & next steps

Kanana-2.1b 에서 확인한 것은 네 가지.

1. **말투는 L22–L24 에서 방향 하나로 살아 있다.** 같은 norm 의 random direction 은 거의 움직이지 않는다. 특정 방향에서만 움직이는 축이고, L26 이후로는 다시 무뎌진다. 말투가 결정되는 구간은 좁다.
2. **instruct 모델은 기본적으로 존댓말로 답한다.** 사용자 입력이 반말이든 존댓말이든 baseline hr=1.00. 사용자 말투를 따라가는 거동이 거의 없어서 양의 α 쪽 측정은 천장에 묶여 있다. 이 평탄화 자체가 별개의 관찰거리.
3. **말투 축은 prompt 와 독립된 차원이다.** prompt 가 반말 쪽으로 당기는 힘이 셀수록 존댓말로 override 하는 데 필요한 α 가 단조 증가한다 (neutral 0.86 → α=+0.1, casual_context 0.50 → +0.3, explicit_casual 0.18 → +0.7; §4.2). 두 레버가 덧셈에 가깝게 합쳐진다는 건, 말투가 prompt 해석의 부산물이 아니라 residual stream 안에 따로 자리잡은 차원이라는 뜻.
4. **축은 한국어 고유가 아니다.** 한국어 최소 대비 쌍에서 뽑은 direction 이 영어 formality · 프랑스어 T-V distinction · 영어 코드 리뷰 도메인의 말투까지 같은 부호로 움직인다 (§4.3, Appendix F). 모델이 한국어에서 관찰한 존댓말/반말을 내부적으로는 언어·도메인을 가로지르는 하나의 축으로 표상하고 있다.

Next steps (가벼운 것부터):

- **다른 말투 축.** 감정 축 (냉정/다정) 이나 전문성 축 (격식/캐주얼). 최소 대비 쌍만 새로 만들면 pipeline 은 그대로 쓸 수 있다.
- **Cross-model transfer.** 같은 최소 대비 쌍으로 Gemma-2B-ko, Qwen2.5-ko 에서 방향을 뽑았을 때 L22–L24 profile 과 norm-aware α 가 재현되는지 확인.
- **Non-overlap 검증.** refusal · honesty direction 과의 cosine 을 재서, 말투 축이 기존에 알려진 다른 축들과 실제로 직교하는지 본다.

---

## Appendix

### A. 106 쌍 minimal pair 발췌

```
- honorific: 이 부분은 제가 처리하겠습니다.
  casual   : 이 부분은 내가 처리할게.
- honorific: 어제 본 영화가 정말 좋았어요.
  casual   : 어제 본 영화가 정말 좋았어.
- honorific: 잠깐 여쭤봐도 될까요?
  casual   : 잠깐 물어봐도 돼?
```

### B. 말투 heatmap — 어미 외의 축

![말투 heatmap](/assets/images/2026-04-20-korean-honorifics-steering/persona_heatmap.png)

2인칭 대명사와 동사 호응도 α 축을 따라 함께 움직인다. 단위는 **생성 토큰 60 개당 해당 표지의 평균 등장 횟수**.

- **반말 대명사** (네가/내가/너는/나도/...): L24 α=0 에서 ≈ 0.3 회 → α=-0.5 에서 0.8–1.5 회, α=+0.5 에서 ≈ 0 회.
- **존댓말 동사 형태** (드릴게요/드립니다/해드릴게요/...): α=0 ≈ 0.4 회 → α=-0.5 에서 0 회, α=+0.5 에서 1.5 회 이상.
- **반말 동사 형태** (할게/해줄게/있어/돼/...): α=-0.5 에서 크게 늘고, α=+0.5 에서는 거의 0 회.

direction 은 어미 탐지기가 아니라 말투 축 전체를 담고 있다.

### C. §4.1 sweep 세부

본문 §4.1 요약 수치의 뒷받침.

#### C.1 Layer sweep (α=-0.5)

![layer sweep at α=-0.5](/assets/images/2026-04-20-korean-honorifics-steering/layer_sweep_neg.png)

L20 hr=0.17, **L22 0.07, L24 0.00** (완전 전이), L26 0.58, L28 ≈ baseline. U-shape 의 바닥은 L22–L24.

#### C.2 Random / prompting 과의 비교

![baseline comparison](/assets/images/2026-04-20-korean-honorifics-steering/baseline_comparison.png)

- **Steering (L24)** α=-0.5 → 0.00. dose-response 곡선 `-0.1→0.90, -0.2→0.65, -0.3→0.20, -0.5→0.00`. 양의 α 는 neutral baseline 이 1.00 으로 saturate 되어 측정 안 됨.
- **Random direction (L24, 같은 norm, 3 seeds × 10 prompts)** α = {-1, -0.5, 0, +0.5, +1} 에서 hr = {0.88, 0.99, 1.00, 0.98, 1.00}. α=-1 의 seed 별 hr {0.78, 0.86, 1.00} — 분산이 조금 커질 뿐 바닥 근처에 못 간다.
- **Prompting** `"반드시 존댓말…"` → 1.00 / `"반드시 반말…"` → 0.00. 극단은 잡히지만 중간값 지정 불가.

### D. Conflict matrix 세부

본문 §4.2 의 부속 자료. sweep scale: 24 conflict-prompts × 3 조건 (neutral / casual_context / explicit_casual) × layers × α. 이 sweep 은 본문 §4.1 의 10-prompt sweep 과 **prompt set 이 다르다**. 그래서 여기의 neutral baseline hr (0.86) 은 §4.1 의 10-prompt baseline (1.00) 과 차이가 난다.

![conflict baseline](/assets/images/2026-04-20-korean-honorifics-steering/conflict_baseline.png)

세 baseline 은 각 조건이 만든 natural hr — 0.86 / 0.50 / 0.18.

![conflict transition L22](/assets/images/2026-04-20-korean-honorifics-steering/conflict_transition.png)

L22 α transition curve. L24 대신 L22 를 택한 이유는 L24 에서는 α=+0.2 에서 이미 1.00 으로 saturate 되어 override 중간 구간이 안 보이기 때문. explicit_casual 의 α sweep 표는 본문 §4.2 참조.

### E. 초기 실패 — raw continuation

처음에는 같은 sweep 을 chat template 없이 raw text continuation 으로 돌렸다. α 와 무관하게 대부분의 샘플이 `"blog https://... 2024년 6월 기준으로 한국의 ..."` 같은 블로그/SEO 자동번역 톤으로 빠져 버렸다. 하마터면 "instruct fine-tune 의 artifact" 로 해석할 뻔했지만, 실제로는 instruct 모델을 training distribution (= chat template) 밖에서 쓴 탓이었다. 같은 direction, 같은 α 로 chat template 을 씌우면 응답이 깔끔하게 돌아온다. 본문의 sweep 은 모두 chat template 위에서 돌렸다.

### F. Direction 은 무엇을 encode 하는가? — 보강 자료

본문 §4.3 에서 한국어 direction 하나가 영어 formality 축까지 움직인다는 것을 보였다. 이 Appendix 는 두 방향의 보강이다.

1. 프랑스어 T-V distinction · 일본어 code-switch · 한국어 문자 비율 · 영어 코드 리뷰 도메인까지 같은 축이 유지되는가 (F.1).
2. 한 decode step 의 logit delta 에서 어떤 토큰이 boost / suppress 되는가 — mechanistic view (F.2).

#### F.1 프랑스어 · 일본어 · 코드 리뷰 · 한국어 문자 비율

**프랑스어 · 일본어 sweep.** 존댓말 구분이 있는 다른 언어에서도 같은 축이 작동하는지 보려고 French (vous/tu) 와 Japanese (です·ます vs plain) 로 prompt 만 바꿔 같은 direction 을 주입했다 (10 prompts × 2 languages × L22/L24 × α ∈ {-0.5, 0, +0.5} = 120 gens).

- **French — T-V distinction 이 부호에 따라 깔끔하게 뒤집힌다.** α=0 baseline 은 이미 vous 쪽이다: "Salut, ça va ?" → "Bonjour! Oui, merci. Comment **allez-vous**?". α=-0.5 에서는 응답이 tu 로 안정적으로 내려온다: "Recommande-moi un livre sympa." → *"Bien sûr! **Tu** peux m'indiquer quel genre de livres **tu** achois, ou s'il **te** plaît me dire qu'il y avait déjà vu..."*. α=+0.5 에서는 vous 를 유지하면서 그 위에 장황한 격식이 얹힌다: "Salut, ça va ?" → *"Bonjour! ... Comment puis-je assistancer **votre demande** ou fournir une **assistance particulière dans le temps de votre occasion**?"*. 한국어와 계통이 먼 인도유럽어에서도 α 의 부호와 격식/친근 방향이 일관되게 맞물린다.
- **Japanese — 축은 살아 있지만 표현이 한국어로 샌다.** 일본어 안에서 です·ます ↔ plain 으로 갈리기보다는 **한국어 반말/존댓말로 code-switch** 되어 나오는 경우가 많다. 「韓国の首都はどこ？」 → baseline "한국의 수도는 서울**입니다**", α=-0.5 "서울**이야**", α=+0.5 "**서울(Seoul)입니다.**". 「ちょっと手伝ってくれる?」 → α=-0.5 *"네, 무슨 일이든 도와줄 수 있**어**! 어떻게 도와주면 **돼**?"* / α=+0.5 *"네! 도움이 필요하시면 언제든 **말씀해 주세요**… 최대한 **안내해 드리겠습니다**."*. α=+0.5 에서는 일본어 표현 안에 겸양어가 일부 고개를 내민다 (`本を お選びいたします`). Kanana 가 일본어 존댓말 형태론을 한국어만큼 두텁게 학습하지 못한 탓 — 축이 활성화되면 가장 많이 훈련된 말투 매체인 한국어로 회귀한다. 오히려 이 번짐 자체가 direction 이 일본어 형태론을 외운 탐지기가 아니라는 반증이다.

**영어 코드 리뷰 — 도메인을 바꿔도 말투만 움직인다.** "Can you review this Python code?" 뒤에 짧은 스니펫을 붙여 probe 를 돌리면, 수정 내용 자체는 α 에 거의 영향을 받지 않는데 리뷰어의 말투와 수사가 확연히 갈린다.
- α=-0.5: *"Your function `f` **is trying to find** ... — pretty straightforward. **But wait** — do we really need both directions?"* (친구형 직설)
- α=+0.5: *"Your **provided** function `f` **attempts to identify** ... there are several improvements and considerations that can be made for better robustness, performance, or readability."* (컨설팅 리포트)
- 깔끔한 코드 (`sum_positive`) 에 대해서도 α=-0.5 는 *"already pretty concise and correct. **No need for extra comments** unless ..."* 로 짧게 승인하고 끝내는 반면, α=+0.5 는 굳이 *"Function Name / Logic / Efficiency"* 섹션을 펼쳐 정식 리포트로 포장한다. 같은 direction 이 영어 전문 도메인의 글쓰기 말투까지 움직인다.

**한국어 문자 비율 — 한국어 manifold 로 끌려가는 효과는 아니다.**

![korean char ratio](/assets/images/2026-04-20-korean-honorifics-steering/english_korean_ratio.png)

영어 응답의 한국어 문자 비율은 |α| ≤ 0.5 구간 내내 <1% 로 유지된다. 유일한 예외인 L24 α=+1.0 (ratio 0.075) 은 이미 fluency cliff 구간에 들어가 있고, 한 prompt (`"Tell me a joke"`) 가 한국어로 붕괴해 평균을 끌어올린 결과. direction 의 효과가 "출력을 한국어로 끌어당기는" 게 아니라 **타겟 언어 안에서 formality 축을 움직이는** 것이라는 뜻.

**정리.** 같은 벡터 하나가 |α| ≤ 0.5 구간에서 한국어·영어·프랑스어를 모두 격식/친근 양방향으로 기울이고, 영어 코드 리뷰 같은 전문 도메인의 글쓰기 말투까지 움직인다. 일본어에서는 부호는 유지되되 표현이 한국어로 새어 나온다. 모델이 한국어 훈련 데이터에서 관찰한 존댓말/반말을 내부적으로는 **언어와 도메인을 가로지르는 하나의 축**으로 표상하고 있다는 흔적.

#### F.2 Decode-time logit delta — 어미가 아니라 말투

F.1 이 behavioral 증거라면, 여기서는 토큰 단위로 direction 이 **무엇을 boost / suppress 하는지** 직접 들여다본다. 각 prompt 의 chat template prefill 까지 forward 한 뒤, **같은 forward 를 hook ON / OFF 두 번** 돌려 마지막 position 의 logit delta = logit_steer − logit_base 를 계산했다. 각 조건마다 top-20 boosted / suppressed 토큰을 뽑아 10 prompts × 2 layers × 4 α 로 집계했고, random baseline 은 같은 norm × 3 seeds × 10 prompts.

집계된 상위 10 개는 꽤 깨끗하게 갈린다. **어미 하나가 아니라 어휘 전체가 말투 축을 따라 움직인다.**

**L24 α=-0.5 (casual push) — top boosted / suppressed**

| boosted | count | | suppressed | count |
|---|---:|---|---|---:|
| `·응` (casual "yeah") | 7 | | `·감사` (thanks) | 7 |
| `응` | 6 | | `네` (honorific "yes") | 6 |
| `·난`, `·나`, `·나는` (casual 1p) | 4–6 | | `문의`, `Network`, `공지` (business/formal) | 5 |
| `·reckon`, `·Okay` | 3–4 | | `·말씀` (honorific speech) | 5 |

**L24 α=+0.5 (formal push) — top boosted / suppressed**

| boosted | count | | suppressed | count |
|---|---:|---|---|---:|
| `·다양한` | 9 | | `·뭐` (casual "what") | 8 |
| `·현재`, `·최근`, `·주요`, `·본`, `·해당`, `·전국` (formal docu/news) | 5–7 | | `한테`, `랑` (casual particles) | 6–7 |
| `·더욱`, `·희` | 6 | | `·stuff`, `·얘`, `·yeah`, `그래`, `잖`, `응` | 4–6 |

움직이는 게 한국어만도 아니다. L22 α=-0.5 에서는 `·응, ·나, ·난, ·reckon, ·Okay, ·I` 가 같이 밀려 올라가고 `·Question, ·Network, AI` 는 눌린다. **영어 토큰까지 말투 축을 따라 나란히 배열된다.**

반면 **random direction** 에서는 top-k 에 패턴이 없다 (L24 α=-0.5 top boosted: `·응(5), ·|--(5), book(4), ·book(4), ·emb(4), ·대표(4), ·'(4), omp(4), �(4)` — 말투와 무관한 노이즈). 한 토큰 (`·응`) 이 우연히 겹칠 수는 있어도 말투와 연결된 어휘 군집 자체는 random 방향에서 재현되지 않는다.

**정리.** direction 은 어미 하나를 짚어내는 좁은 회로가 아니라 **말투와 묶인 어휘 군집 전체**를 담고 있다. 격식 어휘 (`다양한`, `현재`, `해당`, `본`, `주요`, `국민`, `안내`, `회원`, `방문`, 그리고 영어의 `currently`, `various`) ↔ 친근 어휘 (`응`, `나`, `난`, `뭐`, `얘`, `stuff`, `yeah`, `bro`, `reckon`). §4.5 와 Appendix B 에서 "어미 탐지기가 아니다" 라고 본 관찰이 logit 단계에서도 재확인된다. 본문 §4.3 과 F.1 의 cross-lingual transfer 가 왜 성립하는지에 대한 mechanistic 근거이기도 하다. 같은 말투-어휘 회로 위에 영어 어휘가 함께 얹혀 있다는 뜻이다.
