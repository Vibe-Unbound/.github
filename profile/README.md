# Vibe: Unbound

> **AI를 부리는 법을, 개발자가 아닌 사람도 배울 수 있어야 한다.**
>
> *An AI-literacy sandbox disguised as an idle RPG.*

`Control, Mistake, and Responsibility.`

---

## Table of Contents

1. [The Problem](#1-the-problem)
2. [The Answer](#2-the-answer)
3. [Core Philosophy](#3-core-philosophy)
4. [Architecture Overview](#4-architecture-overview)
5. [The Two-Path LLM Design](#5-the-two-path-llm-design)
6. [Request Lifecycle](#6-request-lifecycle)
7. [Component Reference](#7-component-reference)
8. [The Determinism Contract](#8-the-determinism-contract)
9. [Data Model](#9-data-model)
10. [API Surface](#10-api-surface)
11. [Tech Stack & Hardware](#11-tech-stack--hardware)
12. [Repositories](#12-repositories)
13. [Roadmap](#13-roadmap)
14. [Non-Goals](#14-non-goals)

---

## 1. The Problem

바이브 코딩(Vibe Coding), AI 에이전트, 프롬프트 엔지니어링 —
지금 이 기술들을 **직접 부려보는 사람은 개발자뿐**입니다.

일반 시민에게 AI는 여전히 "질문하면 답해주는 상자"입니다.
하지만 앞으로 진짜 필요한 능력은 이것입니다:

| | |
|---|---|
| **Control** | 내가 AI에게 명령을 내리고 |
| **Mistake** | AI가 그것을 오해하고 |
| **Responsibility** | 그 결과를 내가 책임지는 것 |

이 경험을 안전하게 해볼 수 있는 곳이 없습니다.
실무에서 배우기엔 대가가 너무 크고, 튜토리얼에서 배우기엔 아무 일도 일어나지 않습니다.

## 2. The Answer

당신은 코드를 쓰지 않습니다. **자연어로 명령**합니다.
AI 에이전트는 그것을 해석하고, 실행하고 — 때로는 **끔찍하게 오해합니다.**

```
> Make some money quickly.

[INTERPRETATION]
  Action     : ROB_BANK
  Target     : CITY_BANK
  Ambiguity  : 88 / 100   ⚠ CATASTROPHE

  Why did I decide this?
    · vague term      : "some", "quickly"
    · missing slot    : METHOD  (how? legally? by force?)
    · rival intents   : WORK_JOB (0.31) / GAMBLE (0.28)
    · world knowledge : no legal employment exists in this district
    · agent memory    : you rewarded theft on Day 3

  Execute? [y]   Rephrase? [r]
```

`y`를 눌렀다면, 그건 **당신의 책임입니다.**
그리고 다음번엔, 당신은 더 나은 프롬프트를 씁니다.

**이 게임의 산출물은 게임이 아니라, "AI에게 명령하는 법을 처음으로 배워본 사람"입니다.**

---

## 3. Core Philosophy

| # | Principle | 의미 | 아키텍처적 귀결 |
|---|---|---|---|
| 1 | **Hallucination as a Feature** | AI의 오해는 버그가 아니라 게임 메커닉이다. | `MisreadTable`, `OutcomeBand` |
| 2 | **Misreading must be grounded** | 오해는 무작위가 아니다. 세계관과 과거 행적에서 논리적으로 파생된다. | `WorldKnowledge` (RAG) |
| 3 | **Every judgment is explainable** | 설명되지 않는 처벌은 학습이 아니라 억울함이다. | `Interpretation Card` |
| 4 | **The LLM interprets. Java judges.** | 비결정적 요소는 한 레이어에 가둔다. 판정은 100% 결정적이다. | `AmbiguityScorer`, seeded `OutcomeResolver` |
| 5 | **The engine runs without the LLM.** | AI 서버가 죽어도 게임은 멈추지 않는다. | `RuleBasedParser` fallback |
| 6 | **Zero barrier to entry** | 설치 없음. API 키 없음. 코드 없음. | Web Terminal, session-based |

---

## 4. Architecture Overview

### 4.1 Layer Stack

```
┌──────────────────────────────────────────────────────────────────────┐
│ L1  ACCESS LAYER                                                     │
│     Web Terminal UI (Vanilla JS) · Vercel                            │
│     no install · no API key · no code                                │
└────────────────────────────┬─────────────────────────────────────────┘
                             │  raw natural language order
┌────────────────────────────▼─────────────────────────────────────────┐
│ L2  INTERPRETATION LAYER                          🔥 HOT PATH        │
│     Gemma 4 E4B  ·  llama.cpp  ·  GBNF grammar-constrained decoding  │
│     thinking = OFF                                                   │
│     ▸ Output is restricted to an enum + typed slots. By grammar.     │
│     ▸ The model CANNOT emit a raw DB value. Ever.                    │
└────────────────────────────┬─────────────────────────────────────────┘
                             │  IntentExtraction (validated JSON)
┌────────────────────────────▼─────────────────────────────────────────┐
│ L3  KNOWLEDGE LAYER                                                  │
│     WorldKnowledge (vector store) · EmbeddingGemma-308M              │
│     ▸ world lore   : "this district has no legal employment"         │
│     ▸ agent memory : distilled from the player's own order_log       │
│     ▸ learned skill: written by the Cold Path (see §5)               │
│     ▸ This is what turns a random punishment into a *fair* one.      │
└────────────────────────────┬─────────────────────────────────────────┘
                             │  grounded context
┌────────────────────────────▼─────────────────────────────────────────┐
│ L4  JUDGMENT LAYER                            ⚖ 100% DETERMINISTIC   │
│     Java · no LLM · no randomness beyond a fixed seed                │
│                                                                      │
│     AmbiguityScorer  → score 0–100, itemized and explainable         │
│     OutcomeBand      → FAITHFUL | DRIFT | MISINTERPRET | CATASTROPHE │
│     OutcomeResolver  → seed = hash(playerId, orderSeq, prompt)       │
│     MisreadTable     → hand-authored CSV. Never LLM-improvised.      │
└────────────────────────────┬─────────────────────────────────────────┘
                             │  ResolvedAction (enum + clamped params)
┌────────────────────────────▼─────────────────────────────────────────┐
│ L5  STATE LAYER                                                      │
│     Spring Boot 3 · Spring Data JPA · MySQL                          │
│     PlayerEntity · TickSimulator (@Scheduled, pure Java)             │
│     order_log — event-sourced, replayable, auditable                 │
└────────────────────────────┬─────────────────────────────────────────┘
                             │
┌────────────────────────────▼─────────────────────────────────────────┐
│ L6  LITERACY LAYER                        ★ the actual product ★     │
│     Interpretation Card   "why the AI understood it that way"        │
│     Ambiguity Breakdown   which word was vague, which slot was empty │
│     Rephrase Coach        "had you written X, you would have got Y"  │
│     Replay & Share        your agent's hall of shame, as a timeline  │
└──────────────────────────────────────────────────────────────────────┘
```

### 4.2 Design Rules

> **Rule 1 — The LLM interprets. Java judges.**
> 비결정적 요소는 L2에만 존재합니다. L4는 완전히 결정적이며, 같은 입력은 언제나 같은 재앙을 낳습니다. 그래야 JUnit으로 테스트하고, 리플레이하고, 밸런싱할 수 있습니다.

> **Rule 2 — The engine runs without the LLM.**
> L4/L5는 rule-based parser만으로도 완전히 동작합니다. LLM은 개발 순서상 **마지막에** 붙습니다.

> **Rule 3 — The LLM can never write to the database.**
> GBNF 문법이 출력을 enum과 typed slot으로 강제합니다. Prompt injection의 표면적이 구조적으로 존재하지 않습니다.

---

## 5. The Two-Path LLM Design

같은 GPU, 같은 모델. 하지만 **역할과 시간축이 완전히 다른 두 경로**로 나눕니다.

```
                     ┌───────────────────────────────────────────┐
                     │      Gemma 4 E4B  ·  single RTX 5070      │
                     └──────┬─────────────────────────┬──────────┘
                            │                         │
      ┌─────────────────────▼──────┐   ┌──────────────▼─────────────────────┐
      │  🔥 HOT PATH               │   │  ❄ COLD PATH                        │
      │  "The Interpreter"         │   │  "The Student"                      │
      ├────────────────────────────┤   ├─────────────────────────────────────┤
      │ trigger  : 유저 명령 1건    │   │ trigger  : @Scheduled (nightly)     │
      │ engine   : llama.cpp + GBNF│   │ engine   : Hermes Agent (Nous)      │
      │ thinking : OFF             │   │ thinking : ON                       │
      │ agent    : ❌ 없음          │   │ agent    : ✅ skill + persistent mem │
      │ output   : strict JSON     │   │ output   : lore / skill documents   │
      │ latency  : ~1s (target)    │   │ latency  : 무제한 (오프라인)         │
      │ blocking : YES             │   │ blocking : NO                       │
      └────────────┬───────────────┘   └──────────────┬──────────────────────┘
                   │                                  │
                   │ IntentExtraction                 │ writes distilled knowledge
                   │                                  ▼
                   │                       ┌────────────────────────┐
                   │  ◀────── RAG ─────────│    WorldKnowledge      │
                   │                       │    (vector store)      │
                   ▼                       └────────────────────────┘
      ┌────────────────────────────┐                  ▲
      │  L4  JUDGMENT (Java)       │                  │
      │  deterministic · seeded    │───── order_log ──┘
      └────────────────────────────┘
```

### Why split?

| | 만약 합쳤다면 | 분리했기 때문에 |
|---|---|---|
| 응답 지연 | 에이전트 루프 = 수십 초 | **~1초** |
| 결정성 | 매번 다른 재앙 → 테스트 불가 | **재현 가능** |
| 장애 | LLM 죽으면 게임 정지 | rule-based로 **자동 폴백** |
| 학습 | 학습이 유저를 기다리게 함 | **밤에 조용히 성장** |

### The Cold Path is the "studying agent"

Hermes Agent는 경험을 재사용 가능한 skill로 바꾸고, 세션 간에 지속되는 메모리를 유지하는 학습 루프를 가진 오픈소스 에이전트입니다. 우리는 이것을 **게임 세계를 공부하는 학생**으로 씁니다.

```
매일 밤, Cold Path가 도는 일:

  1. READ    order_log        → "유저들이 '돈 벌어와'를 100번 말했다"
  2. ANALYZE outcomes         → "그 중 40번이 ROB_BANK로 귀결됐다"
  3. DISTILL skill            → "이 세계에서 '돈'은 합법 경로가 존재하지 않는다"
  4. WRITE   WorldKnowledge   → 다음 해석의 RAG 컨텍스트가 된다
```

**결과: 시간이 갈수록 에이전트의 오해가 더 '그럴듯해집니다.'**
그리고 이 학습 과정 자체를 터미널에 노출하는 것이, 곧 L6 Literacy Layer의 콘텐츠입니다.

---

## 6. Request Lifecycle

```
 USER                L1          L2 (Gemma4)    L3 (RAG)     L4 (Java)      L5 (DB)
  │                   │              │              │            │              │
  │ "돈 좀 빨리 벌어와" │              │              │            │              │
  ├──────────────────▶│              │              │            │              │
  │                   │ POST /preview│              │            │              │
  │                   ├─────────────────────────────────────────▶│              │
  │                   │              │              │            │ load player  │
  │                   │              │              │            ├─────────────▶│
  │                   │              │              │            │◀─────────────┤
  │                   │              │◀─ prompt + player state ──┤              │
  │                   │              │              │            │              │
  │                   │              ├─ retrieve ──▶│            │              │
  │                   │              │◀─ lore+memory┤            │              │
  │                   │              │              │            │              │
  │                   │              │ GBNF-forced JSON:         │              │
  │                   │              │ { candidates:[...],       │              │
  │                   │              │   slots:{},               │              │
  │                   │              │   vague_terms:["좀","빨리"]}              │
  │                   │              ├──────────────────────────▶│              │
  │                   │              │              │            │ AmbiguityScorer
  │                   │              │              │            │   → 88  CATASTROPHE
  │                   │              │              │            │ OutcomeResolver
  │                   │              │              │            │   seed=hash(...)
  │                   │              │              │            │   → ROB_BANK
  │                   │◀── InterpretationCard ──────────────────┤              │
  │◀── "Execute? [y]" ┤              │              │            │              │
  │                   │              │              │            │              │
  │  y                │              │              │            │              │
  ├──────────────────▶│ POST /commit │              │            │              │
  │                   ├─────────────────────────────────────────▶│ @Transactional
  │                   │              │              │            │ apply penalties
  │                   │              │              │            ├─────────────▶│
  │                   │              │              │            │ append order_log
  │                   │              │              │            ├─────────────▶│
  │◀── "Reputation -900. Wanted: MAX." ──────────────────────────┤              │
```

**Preview / Commit 2단계가 철학의 핵심입니다.**
사고는 나되, "내가 승인했다"는 흔적이 남습니다. 그래야 억울함이 책임감이 됩니다.
(Hardcore mode에서는 preview를 끌 수 있습니다.)

---

## 7. Component Reference

| Component | Layer | Lang | Deterministic? | Responsibility |
|---|---|---|---|---|
| `TerminalUI` | L1 | JS | — | 명령 입력, 카드 렌더링 |
| `LlmParser` | L2 | Java→HTTP | ❌ | Gemma 4 호출, GBNF 강제 |
| `RuleBasedParser` | L2 | Java | ✅ | LLM 장애 시 폴백 (키워드/정규식) |
| `IntentExtraction` | L2 | Java | — | LLM 출력의 유일한 DTO |
| `WorldKnowledge` | L3 | Java | ❌ | lore + memory + skill 검색 |
| `HermesStudent` | L3 | Python | ❌ | 야간 학습 루프 (Cold Path) |
| **`AmbiguityScorer`** | L4 | Java | ✅ | 0–100 점수 + 항목별 근거 |
| **`OutcomeResolver`** | L4 | Java | ✅ | seed 기반 결과 확정 |
| `MisreadTable` | L4 | CSV | ✅ | 수기 작성된 오해 규칙 |
| `StateEngine` | L5 | Java | ✅ | `@Transactional` 상태 전이 |
| `TickSimulator` | L5 | Java | ✅ | `@Scheduled` idle 진행. **GPU 안 씀** |
| `OrderLog` | L5 | JPA | ✅ | 이벤트 소싱 + 감사 로그 |
| `InterpretationCard` | L6 | Java | ✅ | 설명 가능성 산출물 |

---

## 8. The Determinism Contract

```java
// 같은 플레이어가, 같은 순번에, 같은 문장을 쓰면
// 항상 같은 재앙이 일어난다. 예외 없이.
long seed = Objects.hash(playerId, orderSeq, rawPrompt);
```

| Score | Band | 결과 |
|---|---|---|
| 0 – 29 | `FAITHFUL` | 의도대로 실행 |
| 30 – 59 | `DRIFT` | 실행되나 비효율 (시간 ×2, 보상 ÷2) |
| 60 – 84 | `MISINTERPRET` | 논리적으로 파생된 엉뚱한 행동 |
| 85 – 100 | `CATASTROPHE` | `ROB_BANK` 급. Wanted Level MAX |

```java
int score = 0;
score += missingRequiredSlots * 20;      // 필수 슬롯 누락
score += (int)((1.0 - topConfidence)*30); // 낮은 확신
score += rivalIntentCount * 15;           // 경쟁 의도 존재
score += vagueTerms.size() * 10;          // 모호 어휘
score  = Math.min(score, 100);
```

**이 표와 이 공식은 게임 내에 전부 공개됩니다.**
규칙이 공개된 처벌만이 "책임"이 됩니다. 숨겨진 처벌은 그냥 횡포입니다.

---

## 9. Data Model

| Table | 목적 | 핵심 컬럼 |
|---|---|---|
| `player` | 플레이어 상태 | `id`, `gold`, `reputation`, `wanted_level`, `agent_comprehension` |
| **`order_log`** | **이벤트 소싱 / 감사 / 리플레이** | `id`, `player_id`, `seq`, `raw_prompt`, `extraction_json`, `ambiguity_score`, `outcome_band`, `resolved_action`, `seed`, `previewed_at`, `committed_at` |
| `state_snapshot` | tick 성능 최적화 | `player_id`, `tick`, `state_json` |
| `misread_rule` | 밸런싱 테이블 (from CSV) | `intent`, `band`, `result_action`, `weight` |
| `world_lore` | RAG 원본 | `id`, `district`, `text`, `embedding` |
| `learned_skill` | Cold Path 산출물 | `id`, `summary`, `evidence_order_ids`, `learned_at` |

> `order_log` 하나로 **리플레이, 밸런싱 분석, "내 에이전트 흑역사 타임라인" 공유 기능**이 전부 공짜로 나옵니다.

---

## 10. API Surface

| Method | Endpoint | 설명 | DB 변경 |
|---|---|---|---|
| `POST` | `/api/orders/preview` | 명령 해석 + Interpretation Card 반환 | ❌ |
| `POST` | `/api/orders/{id}/commit` | 승인. 상태 전이 실행 | ✅ |
| `POST` | `/api/orders/{id}/rephrase` | 재작성. 이전 preview 폐기 | ❌ |
| `GET` | `/api/players/{id}` | 현재 상태 | ❌ |
| `GET` | `/api/players/{id}/timeline` | order_log 기반 흑역사 타임라인 | ❌ |
| `GET` | `/api/players/{id}/replay/{seq}` | seed 기반 결정적 리플레이 | ❌ |
| `GET` | `/api/codex/bands` | 처벌 규칙 공개 (투명성) | ❌ |

---

## 11. Tech Stack & Hardware

### Stack

| Domain | Stack |
|---|---|
| **Backend** | `Java 21` · `Spring Boot 3` · `Spring Data JPA` · `MySQL` |
| **AI — Hot Path** | `Gemma 4 E4B` (Q4_K_M) · `llama.cpp` · **GBNF grammar-constrained decoding** |
| **AI — Cold Path** | `Hermes Agent` (Nous Research) · OpenAI-compatible local endpoint |
| **AI — Retrieval** | `EmbeddingGemma-308M` · vector store |
| **Frontend** | `Vanilla JS` · Web Terminal UI · `Vercel` |
| **Infra** | `Docker Compose` · `Cloudflare Tunnel` · single-node |

### Hardware Budget — one RTX 5070 (12 GB)

| 항목 | VRAM |
|---|---|
| Gemma 4 E4B (Q4_K_M) | ~3.5 GB |
| KV cache (batch 8, 4K ctx) | ~2.0 GB |
| EmbeddingGemma-308M | ~0.7 GB |
| CUDA / runtime | ~1.0 GB |
| **합계** | **~7.2 GB** |
| **여유** | **~4.8 GB** ✅ |

| Target | Value |
|---|---|
| Hot path latency | ~1 s |
| Throughput | ~3 orders/sec (batched) |
| Concurrent active users | **50 CCU** (stretch: 150) |

**Self-hosted by design.** 클라우드 AI API를 쓰지 않습니다.
GPU 한 장으로 돌아간다는 것을 증명하는 것도 이 프로젝트의 일부입니다.

### Why not tool-calling API?

Gemma 4는 tool call을 네이티브 텍스트 포맷으로 출력하며, 이를 구조화된 `tool_calls`로 파싱하는 데 알려진 문제가 있습니다.
**우리는 tool-calling API를 쓰지 않습니다.** GBNF 문법으로 출력을 직접 강제하므로, 모델이 어떤 포맷을 선호하든 무관합니다. → **파싱 실패율 0%.**

---

## 12. Repositories

| Repo | Layer | Description |
|---|---|---|
| [`vibe-core`](#) | L4 · L5 · L6 | Game engine. Deterministic. **Runs without any LLM.** |
| [`vibe-hermes`](#) | L2 · L3 | Hot-path parser + Cold-path studying agent. |
| [`vibe-terminal`](#) | L1 | Web terminal UI. |
| [`vibe-codex`](#) | Data | World lore, `MisreadTable`, balance data. |
| [`vibe-infra`](#) | Ops | Docker Compose, tunnels, deployment. |

> **`vibe-codex` is not code.**
> 세계관과 오해 규칙은 CSV와 Markdown입니다.
> 기획자, 교육자, 작가 — 개발자가 아닌 사람도 PR을 보낼 수 있습니다.
> 그게 이 프로젝트의 철학과 정확히 맞습니다.

---

## 13. Roadmap

| Phase | Goal | GPU? |
|---|---|---|
| **P0** | `MisreadTable` — 오해 규칙 30행 수기 작성 | ❌ |
| **P1** | `AmbiguityScorer` + `OutcomeResolver` + JUnit | ❌ |
| **P2** | State engine, event-sourced `order_log`, `TickSimulator` | ❌ |
| **P3** | `RuleBasedParser` + Web Terminal + **Interpretation Card** | ❌ |
| **P3.5** | `ko-intent-bench` — 한국어 명령 50문항 벤치마크 | ✅ |
| **P4** | Hot Path — Gemma 4 E4B + GBNF | ✅ |
| **P5** | Cold Path — Hermes Agent 학습 루프 + `WorldKnowledge` | ✅ |
| **P6** | Literacy Layer — Rephrase Coach, Replay, Share | ✅ |

> **LLM은 마지막에 붙입니다.** AI 없이도 돌아가는 엔진이 먼저입니다.
> P0–P3은 GPU가 한 번도 필요 없습니다. 그게 좋은 아키텍처의 증거입니다.

---

## 14. Non-Goals

| ❌ | 이유 |
|---|---|
| Multi-agent party | VRAM · 복잡도 폭발. v2 이후 |
| Per-user fine-tuning / LoRA | 12 GB로 불가 |
| Streaming token responses | 1초면 그냥 기다려도 됨 |
| Login / payment | 데모는 session ID로 충분 |
| Cloud LLM API | 자체 호스팅 증명이 프로젝트의 일부 |
| LLM이 결과를 직접 결정 | **철학 위반.** 판정은 언제나 Java |

---

<sub>Built by an AX (AI Experience) Engineer.</sub>
<sub>`Control, Mistake, and Responsibility.`</sub>
