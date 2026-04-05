---
title: "Beyond the Prompt: Harness Engineering, or Skill Routing"
title_kr: "프롬프트를 넘어서: 하네스 엔지니어링, 다른 말로는 스킬 라우팅"
date: 2026-04-05 12:00:00 +0800
categories:
  - utility-pipeline
excerpt_en: "On a solo game server project, I learned that using AI well depends less on clever prompts and more on building task-specific skills, workflows, and guardrails."
excerpt_kr: "혼자 게임 서버를 만들면서 깨달은 건, AI를 잘 쓰는 핵심이 재치 있는 프롬프트보다 작업별 스킬, 워크플로우, 가드레일을 만드는 데 있다는 점이다."
---

<div class="bilingual">
  <p class="en lede">I am building a game server alone: Go code, dozens of domain modules, concurrency, distributed databases, and operational automation. At this scale, I can no longer imagine working without AI coding agents. But it took me a while to understand that using agents well requires more than good prompts. It requires a good harness.</p>
  <p class="ko lede">게임 서버를 혼자 만들고 있다. Go로 작성된 서버 코드, 수십 개의 도메인 모듈, 동시성 처리, 분산 DB, 운영 자동화. 이 규모의 프로젝트에서 AI 코딩 에이전트 없이 일하는 건 이미 상상할 수 없다. 하지만 에이전트를 잘 쓰려면, "잘 시키는 법"이 필요하다는 걸 깨닫는 데는 시간이 걸렸다.</p>
</div>

<div class="bilingual">
  <p class="en">This post is a summary of what I learned while building and operating a skill system inside Claude Code. It is a story one level above prompt engineering.</p>
  <p class="ko">이 글은 Claude Code에 스킬 시스템을 만들어서 운영하고 있는 경험을 정리한 것이다. "프롬프트 엔지니어링"보다 한 단계 위의 이야기다.</p>
</div>

<div class="bilingual">
  <h2 class="en">When I kept everything inside <code>CLAUDE.md</code></h2>
  <h2 class="ko"><code>CLAUDE.md</code>에 전부 넣었던 시절</h2>
</div>

<div class="bilingual">
  <p class="en">Claude Code has a file called <code>CLAUDE.md</code>. If it sits at the project root, the agent reads it in every conversation. It acts like a working constitution for the repository.</p>
  <p class="ko">Claude Code에는 <code>CLAUDE.md</code>라는 파일이 있다. 프로젝트 루트에 놓으면 에이전트가 매 대화마다 읽는, 일종의 "작업 헌법"이다.</p>
</div>

<div class="bilingual">
  <p class="en">At first, I put every rule there.</p>
  <p class="ko">처음에는 여기에 모든 규칙을 넣었다.</p>
</div>

```text
- Do not write directly to the DB
- Do not mutate shared state from goroutines
- Check the documentation numbering before you write
- Do not misclassify intentional design as a bug during review
- ...
```

<div class="bilingual">
  <p class="en">When there were ten rules, it worked well. The agent followed them, and mistakes went down.</p>
  <p class="ko">규칙이 10개일 때는 잘 작동했다. 에이전트는 규칙을 잘 지켰고, 실수가 줄었다.</p>
</div>

<div class="bilingual">
  <p class="en">The problems started when the list kept growing. Database rules, concurrency rules, documentation rules, review rules, operations rules. They all mattered, so they all landed in the same file. Once the rule count passed fifty, three issues became obvious.</p>
  <p class="ko">문제는 규칙이 많아지면서 시작됐다. DB 관련 규칙, 동시성 규칙, 문서 작성 규칙, 코드 리뷰 규칙, 운영 규칙. 모두 중요했고, 모두 <code>CLAUDE.md</code>에 들어갔다. 규칙이 50개를 넘자 세 가지 문제가 나타났다.</p>
</div>

<div class="bilingual">
  <div class="en">
    <ol>
      <li><strong>Selective neglect.</strong> The agent technically read the whole file, but it missed rules further down surprisingly often.</li>
      <li><strong>Context pollution.</strong> A code review got distorted by operations rules, or operations analysis got polluted by review rules.</li>
      <li><strong>Maintenance hell.</strong> When I changed one rule, I had to manually check whether it conflicted with everything else.</li>
    </ol>
  </div>
  <div class="ko">
    <ol>
      <li><strong>선택적 무시.</strong> 에이전트가 규칙 전체를 "읽기는 하지만" 먼 아래쪽 규칙을 자주 놓쳤다.</li>
      <li><strong>맥락 오염.</strong> 코드 리뷰를 시켰는데 운영 규칙에 영향을 받거나, 그 반대가 발생했다.</li>
      <li><strong>유지보수 지옥.</strong> 규칙 하나를 고치면 다른 규칙과 충돌하는지 사람이 직접 확인해야 했다.</li>
    </ol>
  </div>
</div>

<div class="bilingual">
  <p class="en"><code>CLAUDE.md</code> is excellent for rules that should always apply. It is not a good structure for procedures that only matter during a specific kind of task.</p>
  <p class="ko"><code>CLAUDE.md</code>는 "항상 적용되는 규칙"에는 적합하지만, "특정 작업을 할 때만 필요한 절차"까지 담기에는 구조적으로 한계가 있었다.</p>
</div>

<div class="bilingual">
  <h2 class="en">The idea of skills</h2>
  <h2 class="ko">스킬이라는 개념</h2>
</div>

<div class="bilingual">
  <p class="en">The solution turned out to be simple: make an independent instruction file for each task type, and load it only when needed.</p>
  <p class="ko">해결책은 단순했다. <strong>작업 유형별로 독립된 명령 파일을 만들고, 필요할 때만 로드하는 것.</strong></p>
</div>

<div class="bilingual">
  <p class="en">In Claude Code, if you place Markdown files inside <code>.claude/commands/</code>, you can call them through slash commands like <code>/REVIEW_CODE</code> or <code>/OPS_STATUS</code>. I started calling these files skills.</p>
  <p class="ko">Claude Code에서는 <code>.claude/commands/</code> 디렉터리에 마크다운 파일을 넣으면 슬래시 커맨드(<code>/REVIEW_CODE</code>, <code>/OPS_STATUS</code> 등)로 호출할 수 있다. 이것을 "스킬"이라고 부르기로 했다.</p>
</div>

```text
CLAUDE.md          -> Always loaded. Constitution.
Skill file (.md)   -> Loaded only when invoked. Specialist.
```

<div class="bilingual">
  <p class="en">A single skill looks like this.</p>
  <p class="ko">스킬 하나는 이런 구조다.</p>
</div>

```markdown
# REVIEW_CONCURRENCY - Full concurrency and parallelism audit

## Goal
Inspect the entire Go source tree for Single Writer violations and shared-state access risks inside goroutines

## Detection Types (5)
| Type | Pattern                                      | Risk |
|------|----------------------------------------------|------|
| 1    | Direct write to matchState inside go func    | race |
| 2    | Map key-value assignment inside goroutine    | race |
| 3    | append followed by shared struct reassignment| race |
| ...  |                                              |      |

## Steps
Phase 1: Inventory collection
Phase 2: Block-by-block validation
Phase 3: False-positive regression check
Phase 4: Report generation

## Done When
- [ ] Full inventory collected
- [ ] Every block reviewed
- [ ] Output document generated
```

<div class="bilingual">
  <p class="en">The important part is that a skill makes three things explicit: what to inspect, in what order to do it, and what counts as complete. Telling an agent, "Review concurrency," is not the same as giving it five detection types and a four-phase procedure. The difference in consistency is enormous.</p>
  <p class="ko">핵심은 <strong>스킬이 무엇을 검사하고, 어떤 순서로 수행하고, 언제 완료인지</strong>를 명시한다는 점이다. 에이전트에게 "동시성 리뷰해줘"라고 말하는 것과, 5가지 탐지 유형과 4단계 절차가 적힌 스킬을 실행시키는 것은 결과의 일관성에서 차원이 다르다.</p>
</div>

<div class="bilingual">
  <h2 class="en">Skills turn into layers</h2>
  <h2 class="ko">스킬은 계층이 된다</h2>
</div>

<div class="bilingual">
  <p class="en">Once the skill count passed ten, I naturally needed structure. I now run them in four layers.</p>
  <p class="ko">스킬이 10개를 넘자, 자연스럽게 분류가 필요해졌다. 지금은 4개 계층으로 나눠서 운영하고 있다.</p>
</div>

<div class="bilingual">
  <h3 class="en">Layer 1: Review skills for domain-specific audits</h3>
  <h3 class="ko">1층: 리뷰 스킬, 영역별 전수 점검</h3>
</div>

```text
REVIEW_CODE          Full source code review
REVIEW_CONCURRENCY   Concurrency and parallelism violations
REVIEW_DB            PostgreSQL query and schema audit
REVIEW_SECURITY      Security audit
REVIEW_CONFIG        Three-way config comparison
REVIEW_CHAIN         Interface-to-implementation sync review
REVIEW_GRAFANA       Monitoring dashboard audit
REVIEW_PATH          End-to-end request path integrity check
... 12 in total
```

<div class="bilingual">
  <p class="en">I split one broad request, "Review the code," into twelve specialist skills. Why go that far? Because broad reviews get shallow. If I ask an agent to review the whole codebase, it glances at many things and often misses the concurrency bug or the missing sharding key that actually matters. If I ask it to find only concurrency violations using five named patterns, detection quality goes up dramatically.</p>
  <p class="ko">"코드 리뷰해줘"라는 하나의 요청을 12개의 전문 스킬로 쪼갠 것이다. 왜 이렇게까지 나눴을까? <strong>범위가 넓은 리뷰는 품질이 떨어진다.</strong> "전체 코드 리뷰해줘"라고 하면 에이전트는 모든 걸 조금씩 보다가, 정작 중요한 동시성 버그나 샤딩 키 누락을 놓친다. 반면 "동시성 위반만 찾아줘, 이 5가지 패턴으로"라고 하면 탐지율이 극적으로 올라간다.</p>
</div>

<div class="bilingual">
  <p class="en">Narrow scope plus explicit procedure equals trustworthy output. That is the core rule.</p>
  <p class="ko">좁은 범위 + 명확한 절차 = 신뢰 가능한 출력. 이것이 핵심 원칙이다.</p>
</div>

<div class="bilingual">
  <h3 class="en">Layer 2: Operations skills for health checks and recovery drills</h3>
  <h3 class="ko">2층: 운영 스킬, 상태 확인과 복구 훈련</h3>
</div>

```text
OPS_STATUS     Overall server health judgment and next actions
OPS_DRILL      Recovery rehearsal for outage scenarios
OPS_ROLLBACK   Migration rollback round-trip verification
```

<div class="bilingual">
  <p class="en">If review skills answer, "Is the code correct?", operations skills answer, "Is the server healthy right now?" When I run <code>OPS_STATUS</code>, the agent reads the ops engine, inspects each risk domain, and returns a judgment with recommended actions.</p>
  <p class="ko">리뷰가 "코드가 올바른가?"를 확인하는 거라면, 운영 스킬은 "서버가 건강한가?"를 확인한다. <code>OPS_STATUS</code>를 실행하면 에이전트가 ops engine 코드를 읽고, 각 risk domain의 상태를 판정해서 권장 조치와 함께 보고한다.</p>
</div>

<div class="bilingual">
  <h3 class="en">Layer 3: Workflow skills that force better development habits</h3>
  <h3 class="ko">3층: 워크플로우 스킬, 개발 흐름 강제</h3>
</div>

<div class="bilingual">
  <p class="en">Review and operations skills inspect things. Workflow skills shape the development process itself.</p>
  <p class="ko">리뷰와 운영 스킬이 "점검"이라면, 워크플로우 스킬은 "개발 과정 자체"를 구조화한다.</p>
</div>

<div class="bilingual">
  <div class="en">
    <ul>
      <li><strong>Planning:</strong> Write the implementation plan before a multi-step task starts.</li>
      <li><strong>TDD:</strong> Write the test first, then implement, then refactor.</li>
      <li><strong>Systematic debugging:</strong> Diagnose the bug, form a hypothesis, then verify it.</li>
      <li><strong>Completion validation:</strong> Force a real verification step before declaring the work done.</li>
    </ul>
  </div>
  <div class="ko">
    <ul>
      <li><strong>계획 수립:</strong> 멀티스텝 작업의 구현 계획을 먼저 작성</li>
      <li><strong>TDD:</strong> 테스트를 먼저 쓰고, 구현하고, 리팩터</li>
      <li><strong>체계적 디버깅:</strong> 버그를 만나면 원인 진단 → 가설 수립 → 검증 순서로</li>
      <li><strong>완료 검증:</strong> "다 됐다"고 선언하기 전에 실제 검증 실행을 강제</li>
    </ul>
  </div>
</div>

<div class="bilingual">
  <p class="en">Before I had these skills, the most common failure mode was predictable: the agent patched some code, said, "Fixed," and left the project in a state that did not even build. Workflow skills exist to block that kind of premature completion claim.</p>
  <p class="ko">이 스킬들이 없을 때 가장 자주 벌어지던 일은 뻔했다. 에이전트가 버그를 고치겠다며 코드를 바로 수정하고, "고쳤습니다"라고 말하지만, 실제로는 빌드조차 안 되는 상태. 워크플로우 스킬은 이런 "성급한 완료 선언"을 구조적으로 막는다.</p>
</div>

<div class="bilingual">
  <h3 class="en">Layer 4: Meta skills for maintaining the skills themselves</h3>
  <h3 class="ko">4층: 메타 스킬, 스킬 자체를 관리</h3>
</div>

<div class="bilingual">
  <p class="en">Once the catalog passed twenty-five skills, I needed a skill that maintained the other skills.</p>
  <p class="ko">스킬이 25개를 넘자, 스킬을 관리하는 스킬이 필요해졌다.</p>
</div>

<div class="bilingual">
  <p class="en"><code>HEAL_SKILLS</code> scans every skill file and checks seven kinds of drift, including whether referenced file paths still exist, whether named functions still exist in the codebase, and whether required sections like goal, steps, and completion criteria are still present.</p>
  <p class="ko"><code>HEAL_SKILLS</code>는 모든 스킬 파일을 자동으로 스캔해서 7가지 항목을 점검한다. 스킬이 참조하는 파일 경로가 실제로 존재하는지, 언급하는 함수명이 코드베이스에 아직 있는지, 필수 섹션(목표, 수행 단계, 완료 조건)이 빠지지 않았는지 같은 것들이다.</p>
</div>

<div class="bilingual">
  <p class="en">Refactor the codebase and function names change. Move directories and paths drift. Once a skill goes stale, it starts steering the agent in the wrong direction. <code>HEAL_SKILLS</code> exists to catch that drift before it accumulates.</p>
  <p class="ko">코드를 리팩터링하면 함수명이 바뀌고, 디렉터리 구조가 변하면 경로가 달라진다. 그러면 스킬 안의 참조가 깨진다. 깨진 스킬은 에이전트를 엉뚱한 방향으로 이끈다. <code>HEAL_SKILLS</code>는 이 드리프트를 주기적으로 잡아준다.</p>
</div>

<div class="bilingual">
  <h2 class="en">What practice taught me</h2>
  <h2 class="ko">실전에서 배운 것들</h2>
</div>

<div class="bilingual">
  <p class="en">The theory is the easy part. The practical lessons matter more.</p>
  <p class="ko">이론은 여기까지고, 실제로 운영하면서 부딪힌 문제와 해결책이 더 중요하다.</p>
</div>

<div class="bilingual">
  <h3 class="en">Do not mix reporting and fixing in the same review pass</h3>
  <h3 class="ko">리뷰 2단계 분리: 보고와 수정을 합치면 안 된다</h3>
</div>

<div class="bilingual">
  <p class="en">At first, I let review skills fix issues as soon as they found them. It felt convenient, but it created a tracking problem. If an agent reported ten issues and silently fixed five, I could no longer tell what had actually been found, what had been changed, and what had been skipped.</p>
  <p class="ko">초기에는 리뷰 스킬이 문제를 찾으면 바로 고치게 했다. 편해 보였지만, 심각한 문제가 있었다. 에이전트가 10건을 찾아서 보고하면서 동시에 5건을 수정했다고 하자. 그러면 무엇을 찾았고 무엇을 고쳤는지 추적이 안 된다.</p>
</div>

<div class="bilingual">
  <p class="en">The fix was procedural: Phase 1 generates the report, then the user confirms, then Phase 2 performs the edits. Every review skill now follows that protocol.</p>
  <p class="ko">해결은 절차 분리였다. Phase 1은 보고서 작성, 사용자 확인, 그리고 나서야 Phase 2에서 수정. 지금은 모든 리뷰 스킬이 이 프로토콜을 공유한다.</p>
</div>

<div class="bilingual">
  <h3 class="en">Intent verification: AI often treats “different” as “wrong”</h3>
  <h3 class="ko">의도성 검증: AI는 "다른 것"을 "틀린 것"으로 본다</h3>
</div>

<div class="bilingual">
  <p class="en">The first time I ran the review skills, the agent found thirty-three issues. I was excited. Then I checked them one by one, and about twenty were false positives. Around sixty percent.</p>
  <p class="ko">리뷰 스킬을 처음 실행했을 때 33건의 이슈를 찾았다. 흥분했다. 그런데 하나씩 확인해 보니 20건이 오탐이었다. 60%다.</p>
</div>

<div class="bilingual">
  <p class="en">One example: it flagged a map write inside a goroutine as a race condition. In reality, it was a partitioned-write pattern where each goroutine only touched its own key range. The agent tended to classify unusual patterns as dangerous, even when they were intentional and safe.</p>
  <p class="ko">에이전트가 "이 고루틴 안에서 맵에 쓰고 있으니 race condition입니다"라고 보고한 코드가 실제로는 Partitioned Write, 즉 각 고루틴이 자기 키에만 쓰는 의도된 안전한 패턴인 경우가 있었다. 에이전트는 패턴이 일반적이지 않으면 위험하다고 판단하는 경향이 있다.</p>
</div>

<div class="bilingual">
  <p class="en">So I added an intent-verification protocol to every review skill.</p>
  <p class="ko">해결책은 모든 리뷰 스킬에 <strong>의도성 검증 프로토콜</strong>을 넣는 것이었다.</p>
</div>

```text
If a finding is CRITICAL or WARNING:
1. Trace the call chain two levels deep
2. Search comments, docs, and git blame for intent signals
3. Classify the result as:
   - confirmed_issue
   - intentional_design
   - needs_clarification
```

<div class="bilingual">
  <p class="en">I added one more layer on top of that: false-positive regression checks by a different agent. If the same agent validates its own finding, confirmation bias creeps in. With cross-checking, the false-positive rate dropped from roughly sixty percent to under ten.</p>
  <p class="ko">그리고 한 단계 더, 찾은 에이전트와 다른 에이전트가 교차 검증하는 <strong>오탐 회귀 점검</strong>을 추가했다. 같은 에이전트가 자기가 찾은 것을 검증하면 확증 편향이 생기기 때문이다. 이 과정을 거치면 오탐률이 60%에서 10% 이하로 떨어진다.</p>
</div>

<div class="bilingual">
  <h3 class="en">Shared protocol: make twelve skills speak the same language</h3>
  <h3 class="ko">공통 프로토콜: 12개 스킬이 같은 언어를 쓰게 하기</h3>
</div>

<div class="bilingual">
  <p class="en">Once I had a dozen review skills, the reports started drifting. One used emoji severity levels, another used plain text, another generated tables, another wrote prose. The output became harder to scan and harder to trust.</p>
  <p class="ko">리뷰 스킬이 12개가 되자, 각자 다른 포맷으로 보고서를 만들기 시작했다. 어떤 건 심각도를 이모지로 표시하고, 어떤 건 텍스트로 표시했다. 수정 제안을 테이블로 주는 것도 있고, 산문으로 주는 것도 있었다.</p>
</div>

<div class="bilingual">
  <p class="en"><code>SHARED_REVIEW_PROTOCOL</code> solved that. It defines the shared review contract: intent verification, false-positive regression checks, severity criteria, report format, triage rules, and completion conditions. Each review skill starts with the same requirement.</p>
  <p class="ko"><code>SHARED_REVIEW_PROTOCOL</code>은 이 문제를 해결하기 위해 만든 "리뷰 스킬의 리뷰 규약"이다. 의도성 검증 절차, 오탐 회귀 점검, 심각도 분류 기준, 보고서 포맷, Triage 기준, 완료 조건까지 모든 리뷰 스킬이 공유하는 공통 프로토콜을 하나의 문서에 정의했다.</p>
</div>

```text
Required: Read and apply SHARED_REVIEW_PROTOCOL.md before starting the review.
```

<div class="bilingual">
  <p class="en">It is the same idea as extracting shared logic into a base class.</p>
  <p class="ko">코드에서 공통 로직을 베이스 클래스로 빼는 것과 같은 원리다.</p>
</div>

<div class="bilingual">
  <h2 class="en">What this structure gives me</h2>
  <h2 class="ko">이 구조가 주는 것</h2>
</div>

<div class="bilingual">
  <p class="en">In numbers, it is twenty-five-plus skills, four layers, and one developer. In practice, it gives me three concrete benefits.</p>
  <p class="ko">숫자로 보면 스킬 25개+, 4개 계층, 1인 개발. 체감으로 보면 세 가지가 크다.</p>
</div>

<div class="bilingual">
  <div class="en">
    <ul>
      <li><strong>I stopped forgetting checks.</strong> A concurrency review now reliably runs the same five detection types every time.</li>
      <li><strong>Review quality became stable.</strong> Yesterday's review and today's review use the same criteria and the same format.</li>
      <li><strong>A solo project started to feel like a team.</strong> The coding agent, the review agent, and the operations agent each follow their own process.</li>
    </ul>
  </div>
  <div class="ko">
    <ul>
      <li><strong>"까먹는 게 없어졌다."</strong> 동시성 리뷰를 시키면 항상 5가지 타입을 빠짐없이 검사한다.</li>
      <li><strong>"리뷰 품질이 일정해졌다."</strong> 어제의 리뷰와 오늘의 리뷰가 같은 기준, 같은 포맷으로 나온다.</li>
      <li><strong>"혼자인데 팀처럼 돌아간다."</strong> 코드를 쓰는 나, 리뷰하는 에이전트, 운영 상태를 확인하는 에이전트가 각자의 절차를 따른다.</li>
    </ul>
  </div>
</div>

<div class="bilingual">
  <p class="en">The limits are real too.</p>
  <p class="ko">반면 한계도 분명하다.</p>
</div>

<div class="bilingual">
  <div class="en">
    <ul>
      <li><strong>Skill maintenance is not free.</strong> When the code changes, the skills have to change too.</li>
      <li><strong>Too much procedure kills speed.</strong> Running a four-step review for a five-line patch is wasteful.</li>
      <li><strong>AI compliance is not perfect.</strong> Even if a skill says "do not enter Phase 2 before Phase 1 is complete," the agent sometimes still jumps ahead.</li>
    </ul>
  </div>
  <div class="ko">
    <ul>
      <li><strong>스킬 유지보수가 공짜가 아니다.</strong> 코드가 바뀌면 스킬도 업데이트해야 한다. 그래서 메타 스킬이 필요해진다.</li>
      <li><strong>과도한 절차는 속도를 죽인다.</strong> 5줄 고치는 데 4단계 리뷰를 돌리는 건 비효율이다.</li>
      <li><strong>AI의 절차 준수는 100%가 아니다.</strong> "Phase 1 완료 전에 Phase 2에 진입하지 마라"고 써놔도, 가끔 뛰어넘는다.</li>
    </ul>
  </div>
</div>

<div class="bilingual">
  <p class="en">Skills are not meant to wrap every tiny action. They are there to guarantee consistency on the work that is large enough to deserve it.</p>
  <p class="ko">스킬은 모든 작업에 의무적으로 적용할 도구가 아니다. 큰 작업에서 일관성을 보장하기 위한 장치에 가깝다.</p>
</div>

<div class="bilingual">
  <h2 class="en">How to start</h2>
  <h2 class="ko">시작하려면</h2>
</div>

<div class="bilingual">
  <p class="en">You do not need twenty-five skills on day one. In fact, you probably should not build them.</p>
  <p class="ko">처음부터 25개를 만들 필요 없다. 아니, 만들면 안 된다.</p>
</div>

<div class="bilingual">
  <div class="en">
    <ol>
      <li>Pick one task you repeat the most: code review, pre-commit checks, deployment validation, and so on.</li>
      <li>Write down the rules you have to repeat every single time.</li>
      <li>Turn them into one Markdown file with an order of execution and a definition of done.</li>
      <li>Use it a few times, then update it whenever the agent misses something important.</li>
    </ol>
  </div>
  <div class="ko">
    <ol>
      <li><strong>가장 자주 반복하는 작업 하나</strong>를 골라라. "코드 리뷰", "커밋 전 체크", "배포 확인" 같은 것.</li>
      <li>그 작업을 할 때 <strong>매번 말해야 하는 규칙</strong>을 적어라.</li>
      <li>순서와 완료 조건을 넣어서 <strong>하나의 마크다운 파일</strong>로 만들어라.</li>
      <li>몇 번 써보고, 에이전트가 놓치는 부분을 발견할 때마다 스킬에 추가하라.</li>
    </ol>
  </div>
</div>

<div class="bilingual">
  <p class="en">Once the first skill starts working, the second and third usually follow naturally. If you have a rule that keeps making you think, "Why do I have to say this every time?", that rule is probably your first skill candidate.</p>
  <p class="ko">스킬 하나가 제대로 동작하기 시작하면, 두 번째와 세 번째는 자연스럽게 따라온다. <code>CLAUDE.md</code>에서 "이건 매번 말해야 하는데..." 하고 답답했던 규칙이 있다면, 그게 첫 번째 스킬 후보다.</p>
</div>
