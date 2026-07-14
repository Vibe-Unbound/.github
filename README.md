Vibe: Unbound


AI를 부리는 법을, 개발자가 아닌 사람도 배울 수 있어야 한다.

An AI-literacy sandbox disguised as an idle RPG.




The Problem

바이브 코딩(Vibe Coding), AI 에이전트, 프롬프트 엔지니어링.

지금 이 기술들을 직접 부려보는 사람은 개발자뿐입니다.
일반 시민에게 AI는 여전히 "질문하면 답해주는 상자"입니다.

하지만 앞으로의 세상에서 진짜 필요한 능력은 이것입니다:


내가 AI에게 명령을 내리고
AI가 그것을 오해하고
그 결과를 내가 책임지는 것


이 경험을 안전하게 해볼 수 있는 곳이 없습니다.
실무에서 배우기엔 대가가 너무 크고, 튜토리얼에서 배우기엔 아무 일도 일어나지 않습니다.

The Answer

Vibe: Unbound는 그 경험을 게임으로 옮깁니다.

당신은 코드를 쓰지 않습니다. 당신은 자연어로 명령합니다.
AI 에이전트 HERMES는 그 명령을 해석하고, 실행하고 — 때로는 끔찍하게 오해합니다.

> Make some money quickly.

[HERMES // INTERPRETATION]
  Action     : ROB_BANK
  Target     : CITY_BANK
  Ambiguity  : 88 / 100   ⚠ CATASTROPHE

  Why?
    - vague term      : "some", "quickly"
    - missing slot    : METHOD
    - world knowledge : no legal employment exists in this district
    - agent memory    : you rewarded theft on Day 3

  Execute? [y] / Rephrase? [r]

당신이 y를 눌렀다면, 그건 당신의 책임입니다.
그리고 다음번엔, 당신은 더 나은 프롬프트를 씁니다.

이것이 이 프로젝트의 전부입니다.


Core Philosophy

Control, Mistake, and Responsibility

PrincipleMeaningHallucination as a FeatureAI의 오해는 버그가 아니라 게임 메커닉입니다.Misreading must be justified오해는 무작위가 아닙니다. 세계관과 당신의 과거 행적에서 논리적으로 파생됩니다.Every judgment is explainableAI가 왜 그렇게 이해했는지, 항목별로 전부 공개합니다. 설명되지 않는 처벌은 학습이 아니라 억울함입니다.Zero barrier to entry설치 없음. API 키 없음. 코드 없음. 브라우저와 문장 하나면 됩니다.


Architecture

L1  ACCESS         Web Terminal UI  ·  no install, no key, no code
                          ↓  natural language
L2  INTERPRETATION HERMES — Local LLM 8B, GBNF grammar-constrained
                          ↓  enum + slots only (never raw DB values)
L3  KNOWLEDGE      WorldLore (RAG) + AgentMemory (past orders)
                          ↓  grounded context
L4  JUDGMENT       Java — 100% deterministic
                   AmbiguityScorer · OutcomeResolver · MisreadTable
                   seed = hash(playerId, orderSeq, prompt)
                          ↓  ResolvedAction
L5  STATE          Spring Boot · JPA · MySQL · TickSimulator
                          ↓
L6  LITERACY  ★    Interpretation Card · Ambiguity Breakdown
                   Rephrase Coach · Replay & Share

Design Rules


LLM interprets. Java judges.
비결정적 요소는 L2에만 존재합니다. 판정 로직은 완전히 결정적이며, 같은 입력은 언제나 같은 재앙을 낳습니다. 그래야 테스트하고, 리플레이하고, 밸런싱할 수 있습니다.
The engine runs without the LLM.
L4/L5는 LLM 없이 rule-based parser만으로도 완전히 동작합니다. AI 서버가 죽으면 게임이 멈추는 게 아니라, 조용히 폴백합니다.
The LLM can never write to the database.
HERMES의 출력은 문법으로 강제된 enum과 파라미터뿐입니다. Prompt injection의 표면적이 구조적으로 존재하지 않습니다.



Repositories

RepoLayerDescriptionvibe-coreL4 · L5Game engine. Deterministic. Runs without any LLM.vibe-hermesL2 · L3Intent parser + world knowledge retrieval.vibe-terminalL1 · L6Web terminal UI and the literacy feedback loop.vibe-codexDataWorld lore, misinterpretation tables, balance data.vibe-infraOpsDocker Compose, tunnels, deployment.


vibe-codex is not code.
세계관과 오해 규칙은 CSV와 Markdown입니다. 기획자, 교육자, 작가 — 개발자가 아닌 사람도 PR을 보낼 수 있습니다. 그게 이 프로젝트의 철학과 맞습니다.




Tech Stack

DomainStackBackendJava 21 · Spring Boot 3 · Spring Data JPA · MySQLAI CoreLocal LLM (8B, Q4_K_M) · llama.cpp · GBNF grammar-constrained decodingFrontendVanilla JS · Web Terminal UI · VercelInfraDocker Compose · Cloudflare Tunnel · single RTX 5070 node

Self-hosted by design. 클라우드 AI API를 쓰지 않습니다. GPU 한 장으로 돌아가는 것을 증명하는 것도 이 프로젝트의 일부입니다.


Roadmap

PhaseGoalLLM required?P0MisreadTable — 오해 규칙 테이블 설계❌P1AmbiguityScorer + OutcomeResolver + tests❌P2State engine, event-sourced order_log, tick loop❌P3Rule-based parser + Web Terminal UI❌P4HERMES — LLM parser with grammar constraints✅P5WorldLore RAG — the agent studies the world✅P6Literacy Layer — explanation, coaching, replay✅


LLM은 마지막에 붙입니다. AI 없이도 돌아가는 엔진이 먼저입니다.




What We're Actually Building

이 프로젝트의 산출물은 게임이 아닙니다.

"AI에게 명령하는 법을 처음으로 배워본 사람"입니다.

재미는 후크(hook)일 뿐입니다.


<sub>Built by an AX (AI Experience) Engineer. Control, Mistake, and Responsibility.</sub>
