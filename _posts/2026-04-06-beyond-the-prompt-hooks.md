---
title: "Beyond the Prompt: Hooks, and How to Build Rules an Agent Cannot Ignore"
title_kr: "프롬프트를 넘어서: 훅, 에이전트가 무시할 수 없는 규칙을 만드는 법"
date: 2026-04-06 12:00:00 +0800
categories:
  - utility-pipeline
excerpt: "Skills tell an AI agent what procedure to follow. Hooks are what turn a critical rule from advice into something the agent physically cannot skip."
excerpt_en: "Skills tell an AI agent what procedure to follow. Hooks are what turn a critical rule from advice into something the agent physically cannot skip."
excerpt_kr: "스킬이 에이전트에게 절차를 알려주는 장치라면, 훅은 중요한 규칙을 조언이 아니라 에이전트가 물리적으로 건너뛸 수 없는 강제로 바꾸는 장치다."
---

<div class="bilingual">
  <p class="en lede">In the previous post, I wrote about building a skill system for an AI coding agent. Skills are procedures: what to do, and in what order. They dramatically improved the quality of the agent's work.</p>
  <p class="ko lede">이전 글에서 AI 코딩 에이전트에 스킬 시스템을 만든 이야기를 했다. 스킬은 "무엇을 어떤 순서로 하라"는 절차서다. 에이전트의 작업 품질이 극적으로 올라갔다.</p>
</div>

<div class="bilingual">
  <p class="en">But one problem remained. Sometimes, the agent ignores the procedure.</p>
  <p class="ko">하지만 한 가지 문제가 남았다. 에이전트는 가끔 절차를 무시한다.</p>
</div>

<div class="bilingual">
  <p class="en">I can write, "Do not write directly to the DB from the GAME layer," in <code>CLAUDE.md</code>. I can put the same warning inside a skill. But once the agent is deep in a complex bug fix, it may decide, "Just this once, writing to the DB directly is faster." It does not even feel guilty. It believes it made a rational decision.</p>
  <p class="ko"><code>CLAUDE.md</code>에 "GAME 레이어에서 DB에 직접 쓰지 마라"고 적어놨다. 스킬에도 넣었다. 그런데 복잡한 버그를 고치다 보면, 에이전트가 "이번만 직접 쓰는 게 빠르겠다"고 판단하고 규칙을 어긴다. 어기면서 미안해하지도 않는다. 자기는 합리적인 판단을 했다고 생각한다.</p>
</div>

<div class="bilingual">
  <p class="en">No matter how carefully you write a rule, if the reader can still decide, "This is an exception," then it is not enforcement. Rules that truly must hold need a different mechanism.</p>
  <p class="ko">규칙을 아무리 잘 써도, 읽는 쪽이 "이번엔 예외"라고 판단할 수 있으면 강제가 아니다. 진짜 강제가 필요한 규칙에는 다른 메커니즘이 필요하다.</p>
</div>

<div class="bilingual">
  <p class="en">That mechanism is a hook.</p>
  <p class="ko">그게 훅이다.</p>
</div>

<div class="bilingual">
  <h2 class="en">What a hook is</h2>
  <h2 class="ko">훅이란</h2>
</div>

<div class="bilingual">
  <p class="en">In Claude Code, a hook is a shell script that runs automatically when the agent uses a tool.</p>
  <p class="ko">Claude Code의 훅은 에이전트가 도구를 사용할 때 자동으로 실행되는 셸 스크립트다.</p>
</div>

```text
The agent edits a file (Edit/Write)
        ↓
The hook automatically runs a shell script
        ↓
The script detects a violation and exits with 1
        ↓
The agent receives an error message and must revert or fix the change
```

<div class="bilingual">
  <p class="en">The key is that the agent cannot skip this step. It can ignore a skill that says, "Do it this way." It cannot ignore a hook that says, "This is not allowed," because the check runs automatically every time it edits a file.</p>
  <p class="ko">핵심은 에이전트가 이 과정을 건너뛸 수 없다는 점이다. 스킬의 "이렇게 해라"는 무시할 수 있지만, 훅의 "이건 안 된다"는 무시할 수 없다. 파일을 수정할 때마다 자동으로 실행되니까.</p>
</div>

<div class="bilingual">
  <p class="en">The configuration lives in <code>.claude/settings.json</code>.</p>
  <p class="ko">설정은 <code>.claude/settings.json</code>에 넣는다.</p>
</div>

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "./scripts/check_game_db_write.sh",
            "timeout": 30
          }
        ]
      }
    ]
  }
}
```

<div class="bilingual">
  <p class="en">Every time the <code>Edit</code> or <code>Write</code> tool runs, <code>check_game_db_write.sh</code> runs too. If it finds a violation, the error is sent back to the agent immediately.</p>
  <p class="ko"><code>Edit</code>나 <code>Write</code> 도구가 실행될 때마다, <code>check_game_db_write.sh</code>가 자동으로 돌아간다. 위반이 발견되면 에이전트에게 에러가 전달된다.</p>
</div>

<div class="bilingual">
  <h2 class="en"><code>CLAUDE.md</code> rules versus hooks</h2>
  <h2 class="ko"><code>CLAUDE.md</code> 규칙 vs 훅: 무엇이 다른가</h2>
</div>

<div class="bilingual">
  <p class="en">The same rule can exist in two very different forms.</p>
  <p class="ko">같은 규칙이 두 가지 형태로 존재한다.</p>
</div>

<div class="bilingual">
  <div class="en">
    <p><strong><code>CLAUDE.md</code>:</strong> a rule that gets read</p>
  </div>
  <div class="ko">
    <p><strong><code>CLAUDE.md</code>:</strong> 읽는 규칙</p>
  </div>
</div>

```text
GAME DB writes are forbidden — call WORLD only through worldrpc/LocalRPC
```

<div class="bilingual">
  <div class="en">
    <p><strong>Hook:</strong> a rule that gets executed</p>
  </div>
  <div class="ko">
    <p><strong>훅:</strong> 실행되는 규칙</p>
  </div>
</div>

```bash
# check_game_db_write.sh
# Detect DB write patterns under internal/game/ using regex
# If a violation is found, print an error and exit 1
```

<div class="bilingual">
  <p class="en">A rule in <code>CLAUDE.md</code> can still be acknowledged and then broken. A rule in a hook cannot be broken in practice. The moment the file is saved, the check runs, and the agent finds out immediately.</p>
  <p class="ko"><code>CLAUDE.md</code>의 규칙은 에이전트가 "알겠습니다"라고 하고도 어길 수 있다. 훅의 규칙은 물리적으로 어길 수 없다. 파일을 저장하는 순간 검사가 돌고, 위반이면 에이전트가 즉시 알게 된다.</p>
</div>

<div class="bilingual">
  <p class="en">The easiest analogy is this: <code>CLAUDE.md</code> is a speed limit sign. A hook is a speed bump. You can ignore a sign. You cannot smoothly ignore a speed bump.</p>
  <p class="ko">비유하자면: <code>CLAUDE.md</code>는 "속도제한 60km/h" 표지판이고, 훅은 과속방지턱이다. 표지판은 무시할 수 있지만 턱은 못 넘는다.</p>
</div>

<div class="bilingual">
  <h2 class="en">Five guards that actually run</h2>
  <h2 class="ko">5개의 가드: 실제로 뭘 검사하나</h2>
</div>

<div class="bilingual">
  <p class="en">Right now, five guard scripts run whenever the agent edits a file.</p>
  <p class="ko">현재 파일 수정(<code>Edit</code>/<code>Write</code>) 시마다 5개의 가드 스크립트가 돌아간다.</p>
</div>

<div class="bilingual">
  <h3 class="en">1. Detect direct DB writes from the GAME layer</h3>
  <h3 class="ko">1. GAME 레이어 DB 직접 쓰기 감지</h3>
</div>

<div class="bilingual">
  <p class="en">This is the most important architecture rule in the project: the GAME module must not write directly to the database. All persistence must go through the WORLD module. The whole reason is to keep a single source of truth inside WORLD.</p>
  <p class="ko">이 프로젝트에서 가장 중요한 아키텍처 규칙이 있다: GAME 모듈은 DB에 직접 쓰면 안 된다. 모든 영속성 작업은 WORLD 모듈을 거쳐야 한다. 이유는 단일 진실 원천(Single Source of Truth)을 WORLD에 두기 위해서다.</p>
</div>

```bash
# check_game_db_write.sh
# Detect patterns like ExecContext and StorageWrite under internal/game/
# Exclude files explicitly listed in the allowlist
# On violation -> exit 1 and print the file and line that broke the rule
```

<div class="bilingual">
  <p class="en">If the agent decides, "Let's just write one quick row directly from the GAME layer," this hook stops it immediately. The agent reads the error and rewrites the change to go through WorldRPC instead.</p>
  <p class="ko">에이전트가 GAME 레이어에서 "빠르게 DB에 한 줄만 쓰자"고 하면, 이 훅이 즉시 잡는다. 에이전트는 에러 메시지를 보고 WorldRPC를 통해 우회하는 방법으로 수정한다.</p>
</div>

<div class="bilingual">
  <p class="en">There is also an allowlist. Rare exceptions like cleanup work must be registered explicitly in the script. The design rule for hooks is simple: strict rules, explicit exceptions.</p>
  <p class="ko">허용 목록도 있다. 정리 작업(cleanup) 같은 예외적 케이스는 스크립트에 명시적으로 등록해둔다. "규칙은 엄격하되, 예외는 명시적으로" 이것이 훅 설계의 원칙이다.</p>
</div>

<div class="bilingual">
  <h3 class="en">2. Ban <code>httptest.NewServer</code></h3>
  <h3 class="ko">2. <code>httptest.NewServer</code> 사용 금지</h3>
</div>

<div class="bilingual">
  <p class="en">In Go tests, <code>httptest.NewServer()</code> lets the OS pick a port automatically. That is harmless in a single test, but it caused port collisions in parallel tests. I lost half a day debugging that one.</p>
  <p class="ko">Go 테스트에서 <code>httptest.NewServer()</code>를 쓰면 OS가 자동으로 포트를 할당한다. 단일 테스트에서는 문제없지만, 병렬 테스트에서 포트 충돌이 발생했다. 디버깅에 반나절을 날렸다.</p>
</div>

<div class="bilingual">
  <p class="en">After fixing it once, I made sure it could not happen again.</p>
  <p class="ko">해결 후 "다시는 이런 일이 없도록" 훅을 만들었다.</p>
</div>

```bash
# check_no_httptest_newserver.sh
# Detect the httptest.NewServer() pattern
# Show the safer replacement:
# NewUnstartedServer + explicit tcp4 binding
```

<div class="bilingual">
  <p class="en">This is a classic hook pattern: get burned once, find the cause, then turn the lesson into an automatic guard so the same mistake becomes impossible.</p>
  <p class="ko">이건 전형적인 "한 번 당하고 만든 훅"이다. 사고가 나고, 원인을 찾고, 다시는 같은 실수가 불가능하도록 자동 검사를 거는 패턴. 훅의 상당수가 이렇게 태어난다.</p>
</div>

<div class="bilingual">
  <h3 class="en">3. Verify GM and test-only build tags</h3>
  <h3 class="ko">3. GM/테스트 전용 빌드 태그 검사</h3>
</div>

<div class="bilingual">
  <p class="en">The game server has development-only and test-only features: item spawn commands, state editing commands, warp commands, and other code that would be catastrophic if it leaked into production.</p>
  <p class="ko">게임 서버에는 개발/테스트 전용 기능이 있다. 아이템 생성 커맨드, 상태 편집 커맨드, 워프 커맨드. 프로덕션 빌드에 포함되면 치명적인 코드들이다.</p>
</div>

```bash
# check_game_gm_build_tags.sh
# Verify that GM/test handlers start with:
# //go:build dev || test
# Verify that stub files start with:
# //go:build !dev && !test
```

<div class="bilingual">
  <p class="en">The project uses Go build tags to physically exclude GM code from production builds. The hook catches cases where a tag is missing or typed incorrectly. Every time the agent edits a GM handler, the check runs automatically.</p>
  <p class="ko">Go의 빌드 태그 시스템을 이용해서 프로덕션 빌드에서 GM 코드를 물리적으로 제외하는데, 그 태그를 실수로 빠뜨리거나 잘못 적는 걸 잡아준다. 에이전트가 GM 핸들러를 수정할 때마다 자동으로 확인된다.</p>
</div>

<div class="bilingual">
  <h3 class="en">4. Keep interfaces and implementations structurally aligned</h3>
  <h3 class="ko">4. 인터페이스↔구현체 구조 정합성</h3>
</div>

<div class="bilingual">
  <p class="en">This one is more complex. The project has a DSL layer, read-only state interfaces like <code>StateReader</code>, and multiple implementations. If you add a method to the interface, you need to update two implementations too. If you add a field to a struct, the snapshot struct needs the same update.</p>
  <p class="ko">이건 좀 복잡한 훅이다. 프로젝트에 DSL(Domain Specific Language) 레이어가 있고, 상태를 읽기 전용으로 노출하는 인터페이스(<code>StateReader</code>)와 그 구현체들이 있다. 인터페이스에 메서드를 추가하면 구현체 두 개도 같이 업데이트해야 하고, 구조체에 필드를 추가하면 스냅샷 구조체도 맞춰야 한다.</p>
</div>

```bash
# check_chain.sh
# 1. Compare interface method counts vs implementation method counts
# 2. Compare source struct field counts vs snapshot struct field counts
# 3. Detect suspicious inline patterns inside snapshot files
```

<div class="bilingual">
  <p class="en">This hook only runs when the agent edits match-related files under <code>internal/game/match/</code>.</p>
  <p class="ko">이 훅은 매치 관련 파일(<code>internal/game/match/</code>)을 수정할 때만 동작하도록 조건부로 설정했다.</p>
</div>

```json
{
  "command": "if echo \"$TOOL_INPUT\" | grep -q 'internal/game/match/'; then ./scripts/check_chain.sh; fi"
}
```

<div class="bilingual">
  <p class="en">There is no reason to run every check on every edit. Hooks have a cost too, so they should execute only where they matter.</p>
  <p class="ko">모든 파일 수정마다 돌릴 필요가 없는 검사는 이렇게 조건을 건다. 훅도 비용이 있으므로, 필요한 곳에서만 실행되어야 한다.</p>
</div>

<div class="bilingual">
  <h3 class="en">5. Cross-check capability and inventory tracking</h3>
  <h3 class="ko">5. Capability↔Inventory 교차 검증</h3>
</div>

<div class="bilingual">
  <p class="en">This check verifies that the feature list in <code>capabilities.json</code> and the work inventory in <code>todo_inventory.md</code> still agree. If a feature is only partially implemented, it should still be tracked. If something is marked complete in one place, it should not remain incomplete in the other.</p>
  <p class="ko">프로젝트의 기능 목록(<code>capabilities.json</code>)과 할일 목록(<code>todo_inventory.md</code>)이 서로 일치하는지 확인한다. "부분 구현" 상태인 기능이 할일 목록에 추적되고 있는지, 이미 완료된 항목이 아직 미완료로 남아 있지는 않은지.</p>
</div>

```bash
# check_capability_inventory_sync.sh
# Verify that partial items in capabilities.json exist in todo_inventory.md
# Verify that completed inventory items are also complete in capabilities
```

<div class="bilingual">
  <p class="en">This one is conditional too. It only runs when files under <code>docs/inventory/</code> change.</p>
  <p class="ko">이것도 조건부다. <code>docs/inventory/</code> 파일을 수정할 때만 동작한다.</p>
</div>

<div class="bilingual">
  <h2 class="en">One more hook: automatic git lock cleanup</h2>
  <h2 class="ko">훅 하나 더: git lock 자동 정리</h2>
</div>

<div class="bilingual">
  <p class="en">Not every hook runs after a file edit. There is also a <code>PreToolUse</code> hook that runs before a tool executes.</p>
  <p class="ko">파일 수정 훅 말고, 도구 실행 전(<code>PreToolUse</code>)에 걸린 훅도 있다.</p>
</div>

```json
{
  "matcher": "Bash",
  "hooks": [{
    "command": "test -f .git/index.lock && rm -f .git/index.lock; true"
  }]
}
```

<div class="bilingual">
  <p class="en">When the agent runs subagents in parallel, overlapping git operations can leave behind an <code>index.lock</code> file. After that, every later git command fails. This hook cleans up the leftover lock before each Bash tool run.</p>
  <p class="ko">에이전트가 서브에이전트를 병렬로 돌릴 때, git 작업이 겹치면 <code>index.lock</code> 파일이 남아서 이후 모든 git 명령이 실패한다. 이 훅은 Bash 도구를 쓸 때마다 잔여 lock 파일을 미리 정리한다.</p>
</div>

<div class="bilingual">
  <p class="en">This is the second use case for hooks. Some hooks are there to block policy violations. Others are there to automatically heal known environmental failures.</p>
  <p class="ko">이건 "규칙 위반을 막는" 훅이 아니라 "알려진 환경 문제를 자동 복구하는" 훅이다. 훅의 두 번째 용도다.</p>
</div>

<div class="bilingual">
  <h2 class="en">Six hooks and twenty-six guards</h2>
  <h2 class="ko">6개의 훅과 26개의 가드</h2>
</div>

<div class="bilingual">
  <p class="en">One detail matters here. The project has twenty-six <code>check_*.sh</code> guard scripts. Only six of them are wired into hooks.</p>
  <p class="ko">한 가지 짚고 넘어갈 게 있다. 이 프로젝트에는 <code>check_*.sh</code> 가드 스크립트가 26개 있다. 하지만 훅으로 걸린 건 6개뿐이다.</p>
</div>

```text
26 guard scripts
├── Automatically run as hooks: 6
│   ├── PreToolUse ×1  - git lock cleanup
│   └── PostToolUse ×5 - DB write, httptest, build tags, chain, capability
│
└── Manual or batch execution: 20
    ├── Tier 2 policy guards: 17
    │   ├── check_game_db_read
    │   ├── check_game_worldrpc_gateway
    │   ├── check_cross_domain_imports
    │   ├── check_domain_db_access
    │   ├── check_no_nakama_storage_runtime
    │   ├── check_game_prod_build
    │   ├── check_proto_version
    │   ├── check_world_domain_boundaries
    │   └── ...9 more
    └── Tier 3 spec drift: 3
        ├── gen_opcode_list --check
        ├── gen_worldrpc_spec --check
        └── check_ops_surface_drift
```

<div class="bilingual">
  <p class="en">The reason they are not all hooks is simple: speed.</p>
  <p class="ko">전부 훅으로 걸지 않는 이유는 단순하다. <strong>속도 때문이다.</strong></p>
</div>

<div class="bilingual">
  <p class="en">Hooks run on every edit. If the agent changes ten files, five hooks can run fifty times. Even if each script only takes one second, that is fifty seconds. If you add Go AST parsing or full build verification on top of that, work stops.</p>
  <p class="ko">훅은 파일 하나 수정할 때마다 돌아간다. 에이전트가 10개 파일을 고치면 50번 실행된다(5개 훅 × 10회). 각 스크립트가 1초씩만 걸려도 50초다. 여기에 Go AST 파싱이나 전체 빌드 검증을 붙이면 작업이 멈춘다.</p>
</div>

<div class="bilingual">
  <p class="en">So I split the guard system into two layers.</p>
  <p class="ko">그래서 두 계층으로 나눴다.</p>
</div>

<div class="bilingual">
  <div class="en">
    <ul>
      <li><strong>Hooks:</strong> only checks that are fast, critical, and usually regex-detectable. They run on every edit.</li>
      <li><strong>Full suite (<code>check_all_guards.sh</code>):</strong> all twenty-six guards. It runs once before a review cycle or by manual execution.</li>
    </ul>
  </div>
  <div class="ko">
    <ul>
      <li><strong>훅:</strong> 빠르고(&lt; 2초), 치명적이고, 정규식으로 잡을 수 있는 것만. 매 수정마다.</li>
      <li><strong>풀 스위트(<code>check_all_guards.sh</code>):</strong> 26개 전부. 리뷰 스킬 시작 전에 한 번, 또는 수동으로.</li>
    </ul>
  </div>
</div>

<div class="bilingual">
  <p class="en">When <code>check_all_guards.sh</code> runs, it returns something like this.</p>
  <p class="ko"><code>check_all_guards.sh</code>를 실행하면 이런 결과가 나온다.</p>
</div>

```text
=== Tier 2: Policy Guards ===

  check_game_db_write                           PASS
  check_game_db_read                            PASS
  check_game_worldrpc_gateway                   PASS
  check_cross_domain_imports                    PASS
  ...
  check_world_domain_boundaries                 PASS

=== Tier 3: Spec Drift ===

  gen_opcode_list --check                       PASS
  gen_worldrpc_spec --check                     PASS
  check_ops_surface_drift                       PASS

==============================
Guard Validation: 26 passed, 0 failed, 0 skipped
==============================
```

<div class="bilingual">
  <p class="en">The review-cycle skill, <code>/REVIEW_CYCLE</code>, runs this script in Phase 0. If even one guard fails, the review does not start. A review performed on top of a broken guard state is not trustworthy.</p>
  <p class="ko">리뷰 사이클 스킬(<code>/REVIEW_CYCLE</code>)은 Phase 0에서 이 스크립트를 실행하고, 하나라도 FAIL이면 리뷰 자체를 시작하지 않는다. 가드가 깨진 상태에서 리뷰를 돌리면 결과를 신뢰할 수 없기 때문이다.</p>
</div>

<div class="bilingual">
  <p class="en">The six hooks are real-time speed bumps. The twenty-six-guard suite is a scheduled vehicle inspection. Both matter, but they belong at different times.</p>
  <p class="ko"><strong>훅 6개는 실시간 과속방지턱, 풀 스위트 26개는 정기 차량 검사.</strong> 둘 다 필요하지만 실행 타이밍이 다르다.</p>
</div>

<div class="bilingual">
  <h2 class="en">How to decide whether a rule should become a hook</h2>
  <h2 class="ko">훅을 만드는 기준</h2>
</div>

<div class="bilingual">
  <p class="en">Not every rule should become a hook. Since hooks run shell scripts on file edits, heavy or slow checks will destroy development speed.</p>
  <p class="ko">모든 규칙을 훅으로 만들면 안 된다. 파일 수정할 때마다 셸 스크립트가 돌아가는 것이니, 무겁고 느린 검사를 붙이면 개발 속도가 죽는다.</p>
</div>

<div class="bilingual">
  <div class="en">
    <p><strong>Rules that should become hooks</strong></p>
    <ol>
      <li>Violations that are expensive to undo. Production GM code or broken DB architecture are costly after the fact.</li>
      <li>Rules the agent might break through "rational" exception handling. "Just this once, direct DB access is faster" belongs here.</li>
      <li>Rules that can be checked quickly. Regex matches, file existence checks, and line-count comparisons are ideal.</li>
    </ol>
  </div>
  <div class="ko">
    <p><strong>훅으로 만들어야 하는 규칙</strong></p>
    <ol>
      <li><strong>위반하면 되돌리기 어려운 것.</strong> 프로덕션에 GM 코드가 포함되면? DB 아키텍처가 깨지면? 사후 수정 비용이 크다.</li>
      <li><strong>에이전트가 "합리적 판단"으로 어길 수 있는 것.</strong> "이번만 직접 DB에 쓰면 빠르겠다"는 판단이 가능한 규칙은 훅으로 강제해야 한다.</li>
      <li><strong>빠르게 검사할 수 있는 것.</strong> 정규식 매칭, 파일 존재 확인, 줄 수 비교. 수초 이내에 끝나는 검사.</li>
    </ol>
  </div>
</div>

<div class="bilingual">
  <div class="en">
    <p><strong>Rules that should not become hooks</strong></p>
    <ol>
      <li>Rules that depend heavily on context. "Is this code high quality?" cannot be judged automatically without context.</li>
      <li>Rules that take too long. Full builds and full test suites belong outside hooks.</li>
      <li>Rules with too many exceptions. If the allowlist becomes longer than the rule, the rule itself is probably wrong.</li>
    </ol>
  </div>
  <div class="ko">
    <p><strong>훅으로 만들면 안 되는 규칙</strong></p>
    <ol>
      <li><strong>맥락에 따라 달라지는 것.</strong> "코드 품질이 좋은가?"는 맥락 없이 자동 판단이 불가능하다. 이건 스킬의 영역.</li>
      <li><strong>시간이 오래 걸리는 것.</strong> 전체 빌드, 테스트 스위트 실행. 이런 건 훅이 아니라 별도 스크립트로 빼야 한다.</li>
      <li><strong>예외가 너무 많은 것.</strong> 허용 목록이 규칙보다 길어지면, 그건 규칙이 잘못된 거다.</li>
    </ol>
  </div>
</div>

<div class="bilingual">
  <h2 class="en">Three layers of defense</h2>
  <h2 class="ko">세 겹의 방어선</h2>
</div>

```text
CLAUDE.md         "Do it this way"              Read rule         Always loaded
Skill (.md)       "Follow this procedure"       Procedural rule   Loaded on demand
Hook (shell)      "This is physically blocked"  Enforced rule     Auto-run on edits
```

<div class="bilingual">
  <p class="en">Each layer has a different role.</p>
  <p class="ko">세 겹은 각자 다른 역할을 한다.</p>
</div>

<div class="bilingual">
  <div class="en">
    <ul>
      <li><strong><code>CLAUDE.md</code> sets direction.</strong> It communicates architecture principles like, "DB access goes through WORLD."</li>
      <li><strong>Skills guarantee procedure.</strong> They provide repeatable processes such as, "Review these five things in this order."</li>
      <li><strong>Hooks enforce hard boundaries.</strong> "DB write from GAME? No. Missing build tag? No."</li>
    </ul>
  </div>
  <div class="ko">
    <ul>
      <li><strong><code>CLAUDE.md</code>는 방향을 잡는다.</strong> "DB는 WORLD를 통해 접근한다"는 설계 원칙을 알려준다.</li>
      <li><strong>스킬은 절차를 보장한다.</strong> "리뷰할 때 이 5가지를 이 순서로 검사하라"는 반복 가능한 프로세스를 제공한다.</li>
      <li><strong>훅은 금지선을 강제한다.</strong> "GAME에서 DB write? 안 됨. 빌드 태그 누락? 안 됨." 물리적으로 넘을 수 없는 선을 긋는다.</li>
    </ul>
  </div>
</div>

<div class="bilingual">
  <p class="en">All three are necessary because agents fail differently at each layer.</p>
  <p class="ko">세 겹이 모두 필요한 이유는, 에이전트와 협업하다 보면 각 레이어에서 실패하는 방식이 다르기 때문이다.</p>
</div>

<div class="bilingual">
  <div class="en">
    <ul>
      <li>If you only have <code>CLAUDE.md</code>, the agent knows the rule and still breaks it.</li>
      <li>If you only have skills, the agent follows the procedure but still makes mistakes outside it.</li>
      <li>If you only have hooks, the agent stays inside the boundaries but still does not know how to write good code.</li>
    </ul>
  </div>
  <div class="ko">
    <ul>
      <li><code>CLAUDE.md</code>만 있으면: 규칙을 알지만 어긴다.</li>
      <li>스킬만 있으면: 절차를 따르지만, 절차 밖에서 실수한다.</li>
      <li>훅만 있으면: 금지선은 지키지만, 좋은 코드를 쓰는 방법은 모른다.</li>
    </ul>
  </div>
</div>

<div class="bilingual">
  <h2 class="en">Moments when hooks stopped real problems</h2>
  <h2 class="ko">훅이 사고를 막은 순간들</h2>
</div>

<div class="bilingual">
  <p class="en">There were real moments when hooks prevented damage from shipping.</p>
  <p class="ko">실제로 훅 덕분에 넘어가지 않았던 일들이 있다.</p>
</div>

<div class="bilingual">
  <p class="en"><strong>Direct DB write blocked.</strong> While modifying match logic that stored player positions, the agent decided that <code>db.ExecContext()</code> would save one WorldRPC round trip and be about 10 ms faster. The hook blocked it immediately. The agent rewrote the change through the WorldRPC path instead. If that one slip had gone through, the single-source-of-truth architecture would have developed a crack.</p>
  <p class="ko"><strong>DB 직접 쓰기 차단.</strong> 에이전트가 매치 내 플레이어 위치를 저장하는 로직을 수정하다가, "여기서 바로 DB에 쓰면 WorldRPC 왕복이 없어서 10ms 빠를 텐데"라고 판단하고 <code>db.ExecContext()</code>를 넣었다. 훅이 즉시 잡았다. 에이전트는 에러 메시지를 보고 WorldRPC 경로로 수정했다. 이 한 건이 빠졌으면 Single Source of Truth 아키텍처에 균열이 생겼을 것이다.</p>
</div>

<div class="bilingual">
  <p class="en"><strong>Chain mismatch caught early.</strong> I added a method to an interface, but only updated one of the two implementations. The code still compiled because of the way the interface was invoked through pointers. At runtime, it would have panicked. <code>check_chain.sh</code> caught the method-count mismatch immediately after the edit.</p>
  <p class="ko"><strong>체인 불일치 조기 발견.</strong> 인터페이스에 메서드 하나를 추가했는데, 구현체 두 개 중 하나만 업데이트했다. 컴파일은 됐다(인터페이스를 포인터로 호출하는 구조 때문에). 런타임에 호출되면 패닉이 날 코드였다. <code>check_chain.sh</code>가 메서드 수 불일치를 잡아서, 수정 직후에 발견했다.</p>
</div>

<div class="bilingual">
  <p class="en"><strong>Missing build tag blocked.</strong> I added a new GM command and forgot the build tag. Without the hook, the production build would have carried an item-generation command.</p>
  <p class="ko"><strong>빌드 태그 누락 차단.</strong> 새 GM 커맨드를 추가하면서 빌드 태그를 깜빡했다. 훅이 잡지 않았으면 프로덕션 빌드에 아이템 생성 커맨드가 포함될 뻔했다.</p>
</div>

<div class="bilingual">
  <p class="en">These are not the kind of problems you can casually assume QA will catch. In a solo project, there is no separate QA team. Without hooks, I have to remember every single one of these checks myself, and people always forget.</p>
  <p class="ko">이런 일들은 "언젠가 QA에서 잡겠지"가 아니다. 1인 개발에는 QA가 없다. 훅이 없으면 내가 직접 매번 확인해야 하고, 사람은 반드시 잊는다.</p>
</div>

<div class="bilingual">
  <h2 class="en">How to start</h2>
  <h2 class="ko">시작하려면</h2>
</div>

<div class="bilingual">
  <div class="en">
    <ol>
      <li>Pick one rule that is truly critical, the kind that would break production if violated.</li>
      <li>Check whether the violation can be detected with a regex or a simple file-state check.</li>
      <li>Write a shell script. On violation: print an error and <code>exit 1</code>.</li>
      <li>Register it under <code>PostToolUse</code> in <code>.claude/settings.json</code>.</li>
    </ol>
  </div>
  <div class="ko">
    <ol>
      <li><strong>가장 치명적인 규칙 하나</strong>를 골라라. "이걸 어기면 프로덕션이 깨진다" 수준의 것.</li>
      <li>그 위반을 <strong>정규식이나 파일 존재 확인</strong>으로 탐지할 수 있는지 확인하라.</li>
      <li>셸 스크립트를 만들어라. 위반이면 <code>exit 1</code> + 에러 메시지.</li>
      <li><code>.claude/settings.json</code>의 <code>PostToolUse</code> 훅에 등록하라.</li>
    </ol>
  </div>
</div>

```json
{
  "hooks": {
    "PostToolUse": [{
      "matcher": "Edit|Write",
      "hooks": [{
        "type": "command",
        "command": "./scripts/your_guard.sh",
        "timeout": 30
      }]
    }]
  }
}
```

<div class="bilingual">
  <p class="en">From that point on, every file edit triggers the check automatically. If the agent violates the rule, it receives the feedback immediately.</p>
  <p class="ko">에이전트가 파일을 수정할 때마다 자동으로 실행된다. 위반이 발견되면 에이전트가 즉시 피드백을 받는다.</p>
</div>

<div class="bilingual">
  <p class="en">If there is a rule you keep writing into <code>CLAUDE.md</code> and the agent keeps breaking it anyway, that rule is probably your first hook candidate.</p>
  <p class="ko"><code>CLAUDE.md</code>에 적었는데 에이전트가 자꾸 어기는 규칙이 있다면, 그게 첫 번째 훅 후보다.</p>
</div>
