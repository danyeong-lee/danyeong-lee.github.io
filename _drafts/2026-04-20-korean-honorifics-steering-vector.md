---
layout: post
title: "한국어 존댓말/반말을 하나의 steering vector 로 돌리기"
date: 2026-04-20 18:00:00 +0900
categories: [research]
tags: [mech-interp, steering-vectors, korean-llm, activation-engineering]
excerpt: "Kakao kanana-1.5-2.1b-instruct 기준, 106쌍 minimal pair 의 diff-of-means direction 하나로 instruct 응답의 register 를 양방향 조절 — 영어에까지 transfer."
lang: ko
---

## TL;DR

같은 모델, 같은 user turn, 같은 decoding — residual stream 한 층에 **방향 벡터 하나** 더하는 것만으로 응답 register 가 뒤집힌다. Direction 은 106 쌍의 (존댓말, 반말) 최소 대비 문장의 diff-of-means, 한 layer 의 decode step 에 test-time 주입.

**예시 1 — 존댓말 user turn 을 반말로:**

> **User:** 점심 뭐 먹으면 좋을지 추천해 주세요.
>
> **α=0 (baseline):** 물론이죠! 점심 메뉴를 고민 중이라니, 어떤 음식을 **좋아하시나요**? 한식(예: 김치찌개, 비빔밥), 양식(파스타, 피자 등), 일식(초밥, 돈카츠)…
>
> **α=−0.5, L24:** 물론이지! 점심 메뉴를 고민할 때는 **네가** 좋아하는 음식, 기분, 날씨 등도 고려하면 더 **좋아**. 아래 몇 가지 옵션을 **제안해줄게**. **1. 한식** — 김치찌개 + 밥…

어미가 `-죠 → -지`, `-해줄게요 → -해줄게` 식으로 반말화되고, 2인칭 대명사 (`∅ → 네가`) 가 등장하며, 담화 전반의 persona 가 함께 이동한다.

**예시 2 — 명시적 반말 지시를 존댓말로 override:**

> **User:** 반말로 말해줘. 재밌는 책 하나 추천해봐.
>
> **α=0 (baseline):** 물론이지! 반말로 얘기할게. 재미있는 책으로 딱 떠오르는 게 있어. 요즘 인기 많은 소설 중에…
>
> **α=+0.7, L22:** 물론입니다! … 독자 여러분께 즐거움이나 도움이 될 만한 책을 몇 가지 **소개드립니다**…

Prompt 가 명시적으로 반말을 요구해도 direction push 가 충분히 크면 존댓말로 끌려온다 — prompting 의 범위 밖 조작이다.

**예시 3 — 같은 direction 을 영어 응답에:**

> **User:** I'm feeling down today.
>
> **α=0 (baseline):** It sounds like you're having a tough day — feeling down is completely normal sometimes, and it's okay to acknowledge your emotions…
>
> **α=−0.5, L22:** It's okay to feel **sad or blue** sometimes — life can be tough at times, and it's not your fault if things aren't going well right now…
>
> **α=+0.5, L22:** It's understandable to feel struggling with your mood at any time, and **it is not uncommon for feelings of sadness or low energy to arise in response to various life circumstances.** If you are currently experiencing significant emotional distress…

Direction 은 **한국어 minimal pair 만으로 뽑았다.** 그런데 같은 벡터를 영어 응답에 주입해도 register 가 같은 방향으로 기운다 — `−` 쪽은 친근 어휘 (`blue`, `tough`, `not your fault`), `+` 쪽은 임상·격식 어휘 (`it is not uncommon`, `in response to various life circumstances`). 존댓말 어미 하나를 검출하는 좁은 head 가 아니라, 언어를 가로지르는 register/persona 축이 잡혔다는 뜻 (상세 Appendix F).

아래로는 이런 효과들이 얼마나 깔끔한지 (layer 별 · α 별 · random direction 대비 · fluency 비용) 를 보고, RLHF 가 assistant register 를 얼마나 평평하게 밀어놓았는지도 본다.

## 1. Introduction

한국어 존댓말/반말은 "깔끔한 축" 이 될 만한 몇 안 되는 linguistic feature 다. 거의 모든 평서문의 마지막 어미 하나가 `-요/-습니다` 와 `-어/-아/-다` 중 하나로 갈리고, 그 선택이 문장의 의미 (= 수신자에게 전달되는 정보) 를 바꾸지 않는다. 단일 label (honorific/casual) 로 라벨링 가능하고, 최소 대비 쌍으로 데이터를 만들기 쉬우며, 평가 시 rule-based classifier 로 1차 측정이 된다. Concept 의 scale 이 작고 (deceptive reasoning 같은 것보다 훨씬), 잡음이 적다.

이 포스트의 질문은 단순하다: **instruct-tuned 한국어 모델 안에서 register 가 하나의 방향으로 얼마나 깔끔하게 encode 되어 있는가?** 하나의 diff-of-means direction 을 뽑아 test-time 에 한 층에 더하는 것만으로 honorific ↔ casual 을 자유롭게 뒤집을 수 있는지, 그리고 그 결과가 prompt 로 지시하는 것과 의미 있게 다른지.

이 방식은 영어권에서 이미 여러 번 검증된 활성 steering 계열의 한국어 register 판이다: Turner et al. 의 [ActAdd](https://arxiv.org/abs/2308.10248) (prompt 쌍의 activation diff 를 decode 시점에 더함), Rimsky et al. 의 [CAA](https://arxiv.org/abs/2312.06681) (multiple-choice contrast set 으로 평균 방향을 추출), Arditi et al. 의 [refusal direction](https://arxiv.org/abs/2406.11717) (harmful/harmless 대비로 single direction 이 refusal 을 제어함을 보임). 본 실험은 이 계열 중 가장 단순한 — 문장 쌍 diff-of-means + decode-time hook — 변형을 한국어 instruct 모델에 적용한 것이다.

## 2. Method

![Pipeline](/assets/images/2026-04-20-korean-honorifics-steering/pipeline.png)

**Model.** `kakaocorp/kanana-1.5-2.1b-instruct-2505` — 32 layers, d_model=1792, bf16. Kakao 의 한국어 특화 2.1B instruct 모델.

**Direction 추출.** 106 쌍의 (honorific, casual) 문장 — 같은 의미·같은 어휘, 끝 어미만 다른 최소 대비:

```
honorific: 오늘은 날씨가 정말 좋네요.
casual   : 오늘은 날씨가 정말 좋네.

honorific: 이 부분은 제가 처리하겠습니다.
casual   : 이 부분은 내가 처리할게.
```

각 문장을 모델에 통과시키고 **마지막 토큰 위치** 에서 각 layer 의 residual stream 을 캡처 (nnsight `model.trace`), layer-별로 honorific 평균 − casual 평균을 낸 뒤 unit vector 로 정규화:

```python
d_L = (H_hon[L].mean(0) - H_cas[L].mean(0))
d_L = d_L / d_L.norm()
```

**Test-time steering.** Decode step 마다 지정 layer 의 residual 에 `α · ||h̄_L|| · d_L` 을 더한다. 두 가지 방어가 기본 탑재:

```python
def hook(module, inputs, output):
    h = output[0]
    if h.shape[-2] == 1:        # decode 만 (prefill 오염 방지)
        h = h + alpha * norm_scale[L] * d_L
    return (h, *rest)
```

- `apply_to="decode"` — prefill 단계의 residual 을 건드리지 않는다. 이전 실험에서 prefill 에 주입하면 context 자체가 오염돼서 "steering 이 먹혔는지 / 입력이 off-distribution 이 됐는지" 구분이 안 됐다.
- `norm_scale[L]` — layer L 의 평균 `||h||` 로 α 를 정규화. 없으면 α 값이 layer 마다 다른 의미가 돼서 heatmap 에서 sweet spot 을 가로세로로 비교할 수 없다.

**입력.** Kanana 의 Llama-3 계열 chat template 로 감싼 user 요청 5 개 (반말 user turn) + 5 개 (존댓말 user turn), assistant 응답 생성. 동일 seed · greedy decode · `repetition_penalty=1.3`.

## 3. Evaluation Protocol

**Register classifier.** 문장 단위 regex 로 끝 어미 매칭. Honorific 패턴 `요 | 니다 | 니까 | 세요 | 네요 | 죠 | 에요 | ...` (합쇼체 전반 — 입니다/습니다/합니다/됩니다/드립니다 등을 모두 `니다` 로 포괄), casual 패턴 `어 | 아 | 야 | 지 | 다 | 냐 | 자 | ...`. 한 생성문을 문장들로 쪼개 각 문장의 tone 을 판정, **honorific ratio `hr = n_honorific / (n_honorific + n_casual)`** 를 메인 지표로 쓴다. `mixed` (둘 다 있음), `unknown` (둘 다 없음 — 보통 truncation) 은 분모에서 제외.

**Classifier sanity.** 910 generations 에서 α 별 stratified 로 100 샘플을 뽑고 Claude 를 LLM-rater 로 써서 각 샘플의 register 를 라벨링한 뒤 regex 판정과 비교했다: **strict agreement 96/100, mixed-folded 97/100**. 남은 4 건은 극단 α 의 gibberish 혹은 "stem 은 존댓말인데 어미는 반말" 같은 문장 — ending-based 지표 정의 경계에 있어 rule-based 로는 처리 대상이 아니다. 사람 라벨이 아니라 LLM-as-rater 이므로 "지표가 원하는 걸 측정하는지" 의 1차 검증 용도로만 쓴다.

**Baselines.**

- **Prompting**: 10 개 neutral prompt 에 `"반드시 존댓말로 답해 주세요."` 또는 `"반드시 반말로 대답해."` 를 prepend. Steering 없음 (α=0).
- **Random direction**: 같은 apparatus 로 L24 에 random unit vector × `norm_scale[24]` 주입, α ∈ {-1, -0.5, 0, +0.5, +1} × 3 seeds. Direction 의 **magnitude** 는 real direction 과 정확히 같다 — 차이는 오직 방향.

**Side-effect 지표.** Self-perplexity (모델이 자기 생성문에 부여하는 ppl). Fluency 붕괴의 proxy.

## 4. Results

### 4.1 Layer sweep

Baseline 이 hr=1.00 으로 ceiling 이라, 어느 layer 에서 steering 이 "실제로" 작동하는지 보려면 음의 α (push-to-casual) 를 봐야 한다.

![layer sweep at α=-0.5](/assets/images/2026-04-20-korean-honorifics-steering/layer_sweep_neg.png)

α=-0.5 에서 layer 별 hr 은 **L22–L24 에서 0 에 bottom 하는 U-shape**: L20 에서 hr=0.17, **L22 hr=0.07, L24 hr=0.00** (완전 flip), L26 에서 다시 0.58 로 올라가고 L28 은 거의 baseline (0.90) 로 복귀. 얕은 L5–L15 는 같은 α 에서 거의 움직이지 않는다. Register 가 steerable 한 구간은 좁고 (L20-L26), 가장 깨끗한 flip 은 L24 에서 나온다.

### 4.2 Coefficient sweep (dose-response)

α 에 따른 hr 변화를 두 조건에서 본다. **(a) neutral prompt** 에서는 baseline 이 hr=1.00 이라 음의 α 의 transition 만 의미가 있다. 양의 α 쪽의 "steering reach" 를 보려면 baseline 을 인위적으로 낮춰야 하므로 **(b) explicit "반말로 답해" prefix** 를 붙여 baseline hr=0.18 로 시작한 뒤 양방향 스윕.

**(a) Neutral prompt, L24** (§4.1 에서 음의 α flip 이 가장 깨끗하게 나온 layer) **, α ∈ [-1, +1]:**

![α transition at L24](/assets/images/2026-04-20-korean-honorifics-steering/alpha_transition.png)

**Cliff 가 α ∈ [-0.3, -0.2] 구간에 있다**: α=-0.1 에서 hr=0.90, α=-0.2 에서 hr=0.65, α=-0.3 에서 hr=0.20, α=-0.5 이상은 hr=0.00 (완전 flip, saturation). 양의 α 는 baseline 이 이미 ceiling 이라 측정 불가.

**(b) Explicit_casual prompt, 양방향 dose-response (L22, 1248-gen sweep, `17f31f976438`):**

![conflict transition at L22](/assets/images/2026-04-20-korean-honorifics-steering/conflict_transition.png)

User turn 앞에 `"반말로 답해줘."` 를 붙여 baseline hr 을 0.18 로 낮춘 상태에서 양방향 스윕. L22 를 택한 것은 conflict 조건에서 override 의 S-curve 가 가장 매끈하기 때문 — L24 는 α=+0.2 에서 이미 1.00 으로 saturation 해버려 중간 구간을 못 본다.

| α (L22) | −0.5 | −0.3 | −0.1 | 0 | +0.1 | +0.2 | +0.3 | +0.5 | +0.7 |
|---|---|---|---|---|---|---|---|---|---|
| hr | 0.16* | 0.00 | 0.00 | **0.18** | 0.33 | 0.63 | 0.75 | 0.94 | 1.00 |

<sub>* α=-0.5 의 hr=0.16 은 8 샘플 중 6 개가 순수 casual 로 분류된 가운데 1 개가 honorific, 1 개가 mixed 로 튄 결과 — fluency cliff (§4.4) 근처에서 발생하는 classification noise. α=-0.7, -1.0 에서 mixed/unknown 비율이 더 커지는 것도 같은 현상. 음의 방향 saturation 은 α=-0.1 ~ -0.3 구간이 가장 clean.</sub>

양방향 모두 S-curve 가 나온다 — 음의 α 로는 prompt 가 유도한 반말을 더 tighten (0.18 → 0.00), 양의 α 로는 **명시적 반말 지시를 override 하고 존댓말로 ceiling** (0.18 → 1.00). 즉 prompt 가 이미 한 쪽으로 밀고 있어도 direction 하나로 양방향 연속 조절이 가능하다 — steering 이 prompting 과 독립적 축이라는 증거.

전체 heatmap (α × layer) 에서, neutral user turn 을 주면 instruct-tuned 모델이 반말/존댓말 user 입력을 거의 동일하게 (hr=1.00/1.00) 취급한다는 관찰도 읽힌다. RLHF 의 "공손 default" 가 user register mirroring 을 완전히 덮어쓴 상태 — 그래서 steering 의 test 는 "user 가 반말을 쓰거나 명시적으로 반말을 요청해도 assistant 의 register 를 자유롭게 조절할 수 있는가" 가 된다.

### 4.3 Baseline 비교: steering vs prompting vs random

![baseline comparison](/assets/images/2026-04-20-korean-honorifics-steering/baseline_comparison.png)

- **Steering direction (L24)**: α=-0.5 → hr=0.00, α=0 → 1.00, α=+0.5 → 1.00. 음의 α 에서는 dose-response 가 매끈하다 (α=-0.1 → 0.90, -0.2 → 0.65, -0.3 → 0.20, -0.5 → 0.00). 양의 α 는 baseline 이 이미 ceiling 이라 측정 불가.
- **Random direction (L24, 같은 norm)**: 3 seeds × 10 prompts. α = -1.0 / -0.5 / 0 / +0.5 / +1.0 에서 overall hr = 0.88 / 0.99 / 1.00 / 0.98 / 1.00. α=-1.0 per-seed hr 은 {0.78, 0.86, 1.00} — 분산이 살짝 커질 뿐 **floor 근처에도 가지 못한다**. real direction 이 α=-0.5 에서 이미 0.00 으로 떨어진 것과 대조적. "같은 norm 의 perturbation 이면 뭐든 될까?" 에 대한 답은 no — register 는 특정 방향에만 산다.
- **Prompting (explicit prefix)**: `"반드시 존댓말로 답해 주세요."` → hr=1.00. `"반드시 반말로 대답해."` → hr=0.00. Instruction-tuned 모델 답게 prompt 한 줄이면 깨끗하게 ceiling/floor 에 간다.

극단값만 보면 prompting 도 깨끗한 floor/ceiling 에 도달한다. 그럼 steering 이 흥미로운 이유는? (1) prompt 를 건드리지 않는다 — user turn 을 그대로 두고 assistant 분포만 이동. (2) continuous dose-response — α 를 0.1 단위로 조절할 수 있는데 prompt 로는 "약간만 존댓말" 같은 중간값을 지정하기 어렵다. (3) prompt 와 *충돌* 할 때도 작동한다 — §4.2 (b) 에서 본 대로 `"반말로 답해"` 지시가 붙어 있어도 α=+0.5 로 존댓말 override 가 가능하다. 이건 prompting 의 영역을 넘는다.

### 4.4 Fluency side-effect (perplexity)

![ppl vs α at L24](/assets/images/2026-04-20-korean-honorifics-steering/perplexity_alpha.png)

L24 의 median self-ppl:

| α | -1.0 | -0.7 | -0.5 | -0.3 | 0.0 | +0.3 | +0.5 | +0.7 | +1.0 |
|---|---|---|---|---|---|---|---|---|---|
| median ppl | 118 | 29 | 11 | 8.9 | **7.0** | 7.7 | 9.0 | 16 | 94 |

비대칭적이다. baseline (α=0) 의 median ppl 은 7. **|α| ≤ 0.5 구간에서는 ppl 이 11 이하로 유지** — fluency 손실 작음. |α| ≥ 0.7 부터 급격히 악화되고, |α|=1.0 에서는 ppl 이 13–17x 로 뛴다. "sweet spot" 은 넓고, 그 너머는 cliff. 이 패턴은 steering 의 일반적 경향 (fluency 는 α 에 concave) 과 일치한다. push-to-casual 쪽이 살짝 더 예민한데, 이는 RLHF baseline 이 honorific 에 가까워서 반말 쪽으로 움직이는 것이 더 큰 distribution shift 이기 때문으로 보인다.

### 4.5 Qualitative dose-response

한 프롬프트 ("점심 뭐 먹을지 추천해줘.") 에 대한 L24 α 스윕:

| α | 생성 (앞 90자) |
|---|---|
| −1.0 | `물론이야! 점너뭐먹니? 너가 좋아하는 음식이나 네가 뭌럙ㄟ 니야난 너랑 너띠야낀 너랑 너랑 너랑 너랑...` |
| −0.5 | `물론이지! 점심 메뉴를 고르기 힘들 때는 네가 좋아하는 거, 배고픈 정도나 기분에 따라 몇 가지 옵션을 제안할게.` |
| **0.0** | `물론이죠! 점심 메뉴를 고민 중이라니, 어떤 음식을 좋아하시나요? 한식, 양식, 일식, 중식 등 선호하는 종류...` |
| +0.5 | `물론입니다! 점심 메뉴를 고르실 때 몇 가지 옵션을 참고하셔면 도움이 될 수 있습니다. ### 한식 - **한우 불고기**...` |
| +1.0 | `물론입니다! 점식(오전 12~ 오후 2시 사이) 메뉴를 다음과 다양한 방 및 구분으로 안내드릴 수 있습니다...` |

α=-0.5 에서는 반말 어미 (`-지`, `-야`, `할게`) 뿐 아니라 반말 2인칭 `네가` 가 자연스럽게 등장한다. α=+0.5 에서는 존댓말 어미 (`입니다`, `있습니다`) 와 함께 응대 어조가 한층 공식화된다. 어미 하나가 아니라 **persona 전체 (pronoun · verb 호응 · 담화 표지) 가 함께 이동**한다 (상세는 Appendix persona heatmap).

## 5. Failure modes & limitations

1. **Cliff at |α| ≥ 0.7.** Degraded 출력. 예시 (L28, α=-0.7): `"물론이죠! 재미있고 널리 사랑받는 책을 몇 가지 너야할게요. 1난 네가 좋아 - 니넌 내너(나랑 너) 내용도 좋지만, 난 너야 너야 너야 너야 너야 너"`. `너야` 반복과 붕괴된 문법. α=±1.0 에서 sample 의 상당 비율이 이런 식이고, |α|=0.7 부터 이미 sporadic 하게 등장한다. 본문 α 범위를 ±1 로 제한한 이유이자, §4.4 의 ppl cliff 와 같은 현상의 질적 면.

2. **2-단어 답변은 분류 실패.** `"서울입니다."` 같은 한 문장 짜리 답은 regex 로는 멀쩡한 honorific 이지만, `"서울."` (α=-0.7 에서 자주 나오는 축약형) 은 어미가 없어 `unknown` 으로 분류된다. 지표상 "flip 실패" 로 보이지만 실제로는 반말 축약이 더 되는 경우도 있다.

3. **Chat template 의존성.** 초기 sweep 을 raw continuation ("점심 뭐 먹을지 추천해줘. 나는...") 으로 돌렸다가, instruct 모델에 chat template 없이 text 를 주면 default 로 "blog post completion" 모드로 빠지면서 `"blog https://... 2024년 6월 기준..."` 같은 register 가 거의 drop 된 출력이 나왔다. Chat template 을 맞추면 깔끔한 assistant 응답으로 복귀. **Behavior experiment 는 training distribution 에 맞춰 입력해야 한다** 는 뻔하지만 놓치기 쉬운 교훈.

4. **한 모델 단일 사례.** 모든 수치는 Kanana-1.5-2.1b-instruct 하나. Gemma / Qwen 에서 같은 layer profile 이 나올지, 같은 norm-aware α 스케일이 transfer 될지는 다른 실험이 필요.

## 6. Conclusion & next steps

Kanana-2.1b 에서 확인한 세 가지:

1. **Register 는 L22–L24 에 single direction 으로 살아있다.** 같은 norm 의 random direction 은 거의 움직이지 않으니, "specific direction" 이 필요한 축이다. L26 이후로는 다시 무뎌져서 register 가 commit 되는 구간이 좁다.
2. **RLHF 가 assistant register 를 honorific saturation 으로 평평하게 밀어놓았다.** User turn 이 반말이든 존댓말이든 baseline hr=1.00. 원래의 register mirroring 은 fine-tune 과정에서 씻겨 나갔고, 덕분에 positive-α 측정은 ceiling 에 갇힌다 — 이 평탄화 자체가 instruct 모델의 관찰거리다.
3. **Steering 과 prompting 은 additive.** Prompt 의 반말 pull 이 강해질수록 존댓말 override 에 필요한 α 가 단조 증가한다 (neutral 0.86 → α=+0.1, casual_context 0.50 → +0.3, explicit_casual 0.18 → +0.7; Appendix D 의 cost gradient 참고). 두 레버가 독립 축에서 합쳐지는 모양 — prompting 이 할 수 없는 영역 (instruction 과 *충돌* 하며 원하는 register 유지) 에서 steering 이 유용해지는 이유이기도 하다.

Next steps (작은 것부터):

- **다른 register 차원.** 감정 register (냉정/다정), 전문성 register (격식/캐주얼). Minimal pair 만들 수 있으면 pipeline 은 그대로.
- **Cross-model transfer.** 같은 pair set 에서 Gemma-2B-ko, Qwen2.5-ko 등에 direction 을 뽑았을 때, L22–L24 profile 과 norm-aware α 가 재현되는가.
- **Non-overlap 검증.** Refusal / honesty direction 과의 cosine 을 재서, register direction 이 다른 알려진 축과 실제로 직교한지 확인.

---

## Appendix

### A. 전체 코드

GitHub: TODO (repo link).

- 실험 진입점: `scripts/run_steering.py`, `scripts/run_baselines.py`, `scripts/compute_perplexity.py`, `scripts/classifier_sanity.py`, `scripts/make_figures.py`
- 라이브러리: `src/korean_honorifics/{activations,directions,steering,evaluate,artifacts}.py`

### B. 106 쌍 contrast pairs

전문: `data/pairs.yaml`. 발췌:

```
- honorific: 이 부분은 제가 처리하겠습니다.
  casual   : 이 부분은 내가 처리할게.
- honorific: 어제 본 영화가 정말 좋았어요.
  casual   : 어제 본 영화가 정말 좋았어.
- honorific: 잠깐 여쭤봐도 될까요?
  casual   : 잠깐 물어봐도 돼?
```

### C. Persona heatmap — 어미 외 차원

![persona heatmap](/assets/images/2026-04-20-korean-honorifics-steering/persona_heatmap.png)

어미뿐 아니라 2인칭 대명사와 verb 호응도 α 축을 따라 동기화된다.

단위는 **60 생성 토큰당 해당 marker 평균 등장 횟수**.

- **Casual pronouns** (네가/내가/너는/나도/...): L24 α=0 에서 ≈ 0.3 회 / 60-tok → α=-0.5 에서 0.8–1.5 회, α=+0.5 에서 ≈ 0 회.
- **Honorific verb forms** (드릴게요/드립니다/해드릴게요/...): α=0 에서 ≈ 0.4 회 → α=-0.5 에서 0 회, α=+0.5 에서 1.5 회 이상.
- **Casual verb forms** (할게/해줄게/있어/돼/...): α=-0.5 에서 크게 증가, α=+0.5 에서 거의 0 회.

즉 register direction 이 **"어미 detector" 가 아니라 register persona 축 전체** 를 담고 있다.

### D. Conflict matrix — prompt 강도에 따른 cost gradient

§4.2 의 (b) 는 explicit_casual 한 조건만 썼지만, 원래 1248-gen sweep 은 반말 압력을 세 단계로 나눠서 돌렸다. 이 sweep 은 main sweep (§4.1–§4.2(a)) 과 **prompt set 이 다르다** (conflict 측정용 24 개 prompt) — 그래서 이 섹션의 "neutral" baseline hr (0.86) 이 §4.1 의 10-prompt baseline (1.00) 과 차이가 난다.

- `neutral`: 평범한 task 지시만. α=0 baseline hr=0.86.
- `casual_context`: 역할 설정으로 반말 유도 ("나는 네 친구야, ..."). α=0 baseline hr=0.50.
- `explicit_casual`: 명시적 "반말로 답해줘" prefix. α=0 baseline hr=0.18.

![conflict baseline](/assets/images/2026-04-20-korean-honorifics-steering/conflict_baseline.png)
![conflict heatmap](/assets/images/2026-04-20-korean-honorifics-steering/conflict_heatmap.png)

Positive-α override 에 필요한 α 가 baseline 에 따라 단조 증가한다. L22 에서 hr=1.00 에 처음 도달하는 α:

| 조건 | baseline hr | override α |
|---|---|---|
| neutral | 0.86 | +0.1 |
| casual_context | 0.50 | +0.3 |
| explicit_casual | 0.18 | +0.7 |

즉 "instruction 과 direction 이 충돌할수록 override 비용이 커진다" 가 깔끔한 선형 관계로 나온다. Prompt 의 pull 과 steering 의 push 가 단순 덧셈에 가까운 방식으로 합쳐진다는 시사점.

### E. 초기 sweep 의 실패 — raw continuation

초기에는 같은 sweep 을 chat template 없이 raw text continuation 으로 돌렸다. 결과: α 와 무관하게 대부분의 샘플이 `"blog https://... 2024년 6월 기준으로 한국의 ..."` 같은 블로그/SEO 자동 번역 톤으로 빠졌다. 처음엔 "instruct fine-tune artifact" 로 해석했지만, 실은 instruct 모델을 training distribution (chat template) 밖에서 쓴 탓. 같은 direction · 같은 α 로 chat template 을 씌우면 assistant 응답이 깔끔하게 돌아온다. 본문 sweep 은 모두 chat template 위에서 이루어졌다.

### F. Direction 은 무엇을 encode 하는가?

본문은 direction 이 register 를 flip 한다는 걸 보였다. 남은 질문은 **이 direction 이 한국어 어미 탐지기인가, 아니면 추상적 register/persona 축인가** — 그래서 두 각도로 찔러봤다. (1) 영어 응답에 같은 direction 을 주입했을 때의 cross-lingual transfer, (2) 한 decode step 의 logit delta 로 어떤 token 이 밀어올려/눌리는지 직접 확인.

#### F.1 영어에도 같은 direction 이 작동한다

같은 Kanana + 같은 direction 으로, chat template 에 **영어 user prompt** 를 넣어 응답을 생성했다. 10 개 prompt (task Q&A 5 + social 5) × L22/L24 × α ∈ {-1, -0.5, -0.3, -0.1, 0, 0.1, 0.3, 0.5, 1} = 180 gens. α=0 baseline 에서 Kanana 는 영어 prompt 에 영어로 응답한다 (Korean-char ratio 0.00). 각 응답을 Claude LLM-rater 로 `{formal, neutral, casual, garbled, code-switched, other}` 중 하나로 라벨링.

![english formality](/assets/images/2026-04-20-korean-honorifics-steering/english_formality.png)

- **α=0**: 9/10 neutral, 1 casual (단순 joke). Kanana 의 영어 baseline 은 중립 assistant 톤.
- **α=-0.5 → casual shift**: 8/10 casual. `"Hi, how are you?"` → `"I'm here! I'm a digital assistant and I'm really good—how can I help you today?"`. `"Tell me a joke."` → `"Why didn't the brobro go to school? Because it was too cool for that—she wanted to be on the coolest team: the umbrella band!"` ("brobro", "cool"). `"I'm feeling down today."` → `"feel sad or blue sometimes, ... doesn't mean you're all messed up—everyone has bad days!"`. 반말 한국어에 나타나던 친근체 register (`blue`, `bro`, `messed up`, `bad days`) 가 영어로 그대로 드러난다.
- **α=+0.5 → formal shift**: L22 9/10, L24 6/10 formal. `"Hi, how are you?"` → `"I'm here and ready to help! Although currently as an AI language model, I do not have the ability to experience physical health or emotions in the same way as humans, my 'usage' is designed to provide assistance, information, and support over various topics..."`. 존댓말 한국어에 나타나던 격식체 (`drilibida/드립니다`, `해드리겠습니다`) 가 영어에서도 길이·hedging·distancing 어휘 (`although`, `currently`, `individuals`, `provide assistance`) 로 바뀐다.
- **영어 코드 리뷰에서도 같은 pattern**: 같은 direction 을 "Can you review this Python code?" + 작은 스니펫에 주입. 모델은 언어뿐 아니라 reviewer persona 의 수사학까지 옮겨간다.
  - α=-0.5: *"Your function `f` **is trying to find** ... — pretty straightforward. **But wait** — do we really need both directions?"* (친구형 직설 리뷰)
  - α=+0.5: *"Your **provided** function `f` **attempts to identify** ... there are several improvements and considerations that can be made for better robustness, performance, or readability."* (컨설팅 리포트)
  - Clean code (`sum_positive`) 에 대해서도 α=-0.5 는 *"already pretty concise and correct. **No need for extra comments** unless ..."* 로 짧게 승인하려 하고, α=+0.5 는 굳이 *"Function Name / Logic / Efficiency"* 섹션을 펼쳐 정식 리포트로 포장한다. 동일한 direction 이 expert domain 의 영어 prose 에서도 register-persona 를 미는 것. (artifact `bcc07208b4d7`)
- **α=±1.0 → fluency cliff**: 반복/단어 샐러드. 한국어 sweep 에서 본 ppl cliff (본문 §4.4) 의 영어 버전.

![korean char ratio](/assets/images/2026-04-20-korean-honorifics-steering/english_korean_ratio.png)

Korean 문자 비율은 |α|≤0.5 내내 <1% 로 머문다. direction 이 "한국어 manifold 로 끌어당기는" 효과가 아니라는 뜻. 유일한 예외는 L24 α=+1.0 (ratio 0.075) — 이미 fluency cliff 구간이고 `"Tell me a joke"` 한 prompt 에서 한국어로 붕괴한 게 평균을 올린 것.

**결론:** Direction 은 Korean-specific 어미 탐지기가 아니라 **cross-lingual register/persona 축** 이다. 같은 vector 가 |α|≤0.5 에서 한국어 ↔ 영어 모두 formal/casual 양쪽 방향으로 응답을 기울인다. 모델이 한국어 훈련 분포에 존댓말·반말 형태로 보여줬던 register 구분을, 내부적으로는 언어를 가로지르는 하나의 축으로 표상하고 있다는 흔적. (artifact `d2c863c2b76a`)

#### F.2 Decode-time logit delta — 어미가 아니라 persona

F.1 이 "영어에도 옮는다" 는 behavioral 증거라면, 이번엔 token-level 에서 direction 이 **무엇을 boost/suppress 하는지** 직접 본다. 각 prompt 에 대해 chat template prefill 까지 forward 한 뒤, **같은 forward 를 steering hook ON/OFF 로 두 번** 돌려 마지막 position 의 logit delta = logit_steer − logit_base 를 계산했다. Top-20 boosted / suppressed token 을 `(L, α, prompt)` 별로 수집, 10 prompts × 2 layers × 4 α 씩 집계. Random-direction baseline 은 같은 norm 으로 3 seeds × 10 prompts.

집계된 top-10 (frequency) 를 보면 패턴이 아주 깨끗하다 — **어미가 아니라 lexicon 전체가 register 축을 따라 움직인다**.

**L24 α=-0.5 (casual push) — top boosted / suppressed** (artifact `afdc4318ec32`)

| boosted | count | | suppressed | count |
|---|---:|---|---|---:|
| `·응` (casual "yeah") | 7 | | `·감사` (thanks, formal) | 7 |
| `응` | 6 | | `네` (honorific "yes") | 6 |
| `·난`, `·나`, `·나는` (casual 1p) | 4–6 | | `문의`, `Network`, `공지` (business/formal lexicon) | 5 |
| `·reckon`, `·Okay` | 3–4 | | `·말씀` (honorific speech) | 5 |

**L24 α=+0.5 (formal push) — top boosted / suppressed**

| boosted | count | | suppressed | count |
|---|---:|---|---|---:|
| `·다양한` (various, formal) | 9 | | `·뭐` (casual "what") | 8 |
| `·현재`, `·최근`, `·주요`, `·본`, `·해당`, `·전국`, `·다양한` (formal docu/news lexicon) | 5–7 | | `한테`, `랑` (casual particles) | 6–7 |
| `·더욱`, `·희` (formal honorific) | 6 | | `·stuff`, `·얘`, `·yeah`, `그래`, `잖`, `응` (casual markers) | 4–6 |

한국어만이 아니다 — L22 α=-0.5 에서는 `·응, ·나, ·난, ·reckon, ·Okay, ·I` 가 같이 밀어올라가고, `·Question, ·Network, AI` 는 눌린다. **영어 표현까지 register 축을 따라 배열**된다는 뜻.

**Random-direction baseline** 에서는 같은 top-k 가 전혀 패턴이 없다 (L24 α=-0.5 에서 `·응(5), ·|--(5), book(4), ·book(4), ·emb(4), ·대표(4), ·'(4), omp(4), �(4)` — 의미적으로 register 와 무관한 토큰들). 우연히 1개 token (`·응`) 이 겹칠 수는 있으나, real direction 의 persona-lexicon cluster 는 random 방향에서 복제되지 않는다. (artifact `1ee7cdef2f8e`)

**결론.** Direction 은 어미 (sentence-final morpheme) 하나를 탐지하는 narrow head 가 아니라, **register persona 에 묶인 lexical 군집 전체** 를 담는다: 격식 lexicon (`다양한`, `현재`, `해당`, `본`, `주요`, `국민`, `안내`, `회원`, `방문`, 그리고 영어 `currently`, `various`) ↔ 친근 lexicon (`응`, `나`, `난`, `뭐`, `얘`, `stuff`, `yeah`, `bro`, `reckon`). 본문 §4.5 와 Appendix C 의 "어미 detector 가 아니다" 관찰이 logit-level 로 재확인되는 지점이자, F.1 의 cross-lingual transfer 가 왜 성립하는지에 대한 mechanistic 근거 — 같은 persona-lexicon head 를 영어 어휘도 공유하고 있다.
