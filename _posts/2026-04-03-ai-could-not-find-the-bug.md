---
title: AI Wrote the Code, but Could Not Find the Bug
title_kr: AI가 코드를 다 짰는데 버그를 못 찾았다
date: 2026-04-03 12:00:00 +0800
categories:
  - server-engineering
excerpt_en: AI helped me build and review an MMORPG server, but two serious bugs only surfaced after 6,216 simulation runs and well over 100 load tests.
excerpt_kr: AI와 함께 MMORPG 서버를 만들었지만, 진짜 문제는 6,216회의 시뮬레이션과 100회가 넘는 부하 실험 뒤에야 드러났다.
---

<div class="bilingual">
  <p class="en lede">AI coding tools are very good at writing code that looks correct. They are also surprisingly good at review. What they still cannot tell me is whether a live server will behave correctly when thousands of sessions start colliding in real time. This post is a record of how I ended up finding two bugs only after 6,216 simulation runs and well over 100 load tests.</p>
  <p class="ko lede">AI 코딩 도구는 겉보기에 올바른 코드를 만드는 데 정말 능하다. 코드 리뷰도 꽤 잘한다. 그런데 수천 개의 세션이 실제 런타임에서 동시에 부딪히기 시작할 때 서버가 어떻게 행동할지는 말해주지 못한다. 이 글은 6,216회의 시뮬레이션과 100회가 넘는 부하 실험 끝에야 두 개의 버그를 잡아낸 기록이다.</p>
</div>

<div class="bilingual">
  <h2 class="en">I was building the server with AI in the loop</h2>
  <h2 class="ko">AI와 함께 서버를 만들고 있었다</h2>
</div>

<div class="bilingual">
  <p class="en">I am building an MMORPG server mostly on top of Nakama, with Go plugins carrying the game-specific logic. I use AI coding tools aggressively. They help write code, review it, and clean it up. In the local sense, they are excellent. Types pass. Tests pass. Small edge cases get caught.</p>
  <p class="ko">나는 Nakama 위에 Go 플러그인을 얹는 구조로 MMORPG 서버를 만들고 있다. 개발 과정에서 AI 코딩 도구도 적극적으로 쓴다. 코드를 쓰고, 리뷰하고, 정리하는 데 큰 도움이 된다. 로컬한 관점에서는 정말 훌륭하다. 타입 체크도 통과하고, 테스트도 통과하고, 작은 edge case도 잘 잡아낸다.</p>
</div>

<div class="bilingual">
  <p class="en">The problem is that local correctness is not the same thing as system behavior. A server can look clean in a diff and still fail in ugly ways once Redis state, presence streams, WebSocket lifecycles, retries, and load spikes begin interacting at the same time.</p>
  <p class="ko">문제는 로컬한 정확성이 곧 시스템 동작을 의미하지는 않는다는 점이다. diff 상으로는 깔끔해 보여도, Redis 상태와 presence stream, WebSocket lifecycle, retry, 부하 급증이 동시에 얽히기 시작하면 서버는 아주 지저분한 방식으로 무너질 수 있다.</p>
</div>

<div class="bilingual">
  <h2 class="en">The first signal was upside down</h2>
  <h2 class="ko">첫 번째 이상 징후는 직관과 정반대였다</h2>
</div>

<div class="bilingual">
  <p class="en">The admission gate on the server was behaving backward. If I raised the session cap per node, connection errors were supposed to go down. Instead, they exploded.</p>
  <p class="ko">서버의 admission gate가 이상하게 움직이고 있었다. 노드당 세션 cap을 올리면 connection error가 줄어야 맞다. 그런데 실제로는 반대로 폭증했다.</p>
</div>

<div class="table-wrap bilingual">
  <div class="en">
    <table>
      <thead>
        <tr>
          <th>Cap</th>
          <th>Connection Errors</th>
          <th>What Changed</th>
        </tr>
      </thead>
      <tbody>
        <tr>
          <td>8,000</td>
          <td>baseline</td>
          <td>-</td>
        </tr>
        <tr>
          <td>12,000</td>
          <td>+188%</td>
          <td>Cap up 50%, errors up 3x</td>
        </tr>
        <tr>
          <td>16,000</td>
          <td>+4,185%</td>
          <td>Cap up 2x, errors up 42x</td>
        </tr>
      </tbody>
    </table>
  </div>
  <div class="ko">
    <table>
      <thead>
        <tr>
          <th>Cap</th>
          <th>Connection Errors</th>
          <th>변화</th>
        </tr>
      </thead>
      <tbody>
        <tr>
          <td>8,000</td>
          <td>baseline</td>
          <td>-</td>
        </tr>
        <tr>
          <td>12,000</td>
          <td>+188%</td>
          <td>Cap 50% 증가, errors 3배</td>
        </tr>
        <tr>
          <td>16,000</td>
          <td>+4,185%</td>
          <td>Cap 2배 증가, errors 42배</td>
        </tr>
      </tbody>
    </table>
  </div>
</div>

<div class="bilingual">
  <p class="en">I asked AI to review the admission gate code and explain why higher caps could lead to more errors. It came back with plausible suspects: stale heartbeat counts, Redis races, garbage collection timing. All of them sounded reasonable. None of them were the real cause.</p>
  <p class="ko">나는 AI에게 admission gate 코드를 다시 읽히고, 왜 cap을 높였는데 errors가 늘 수 있는지 물었다. AI는 그럴듯한 후보들을 제시했다. stale heartbeat count, Redis race, garbage collection timing. 전부 말은 됐다. 하지만 진짜 원인은 아니었다.</p>
</div>

<div class="bilingual">
  <p class="en">That was the moment I had to admit something simple: static analysis can tell me whether the code is locally coherent. It cannot reliably tell me what happens when eight thousand sessions begin retrying, drifting, and colliding inside a real runtime.</p>
  <p class="ko">그 지점에서 아주 단순한 사실을 인정해야 했다. 정적 분석은 코드가 로컬하게 일관적인지는 말해줄 수 있다. 하지만 8,000개의 세션이 실제 런타임에서 retry하고, 엇갈리고, 서로 충돌하기 시작할 때 무슨 일이 벌어지는지는 믿을 만하게 말해주지 못한다.</p>
</div>

<div class="bilingual">
  <h2 class="en">I stopped reading code and started running experiments</h2>
  <h2 class="ko">코드를 더 읽는 대신 실험을 돌리기 시작했다</h2>
</div>

<div class="bilingual">
  <p class="en">At that point I had two options. I could keep reading more code and keep asking more speculative questions. Or I could treat the server like a systems problem: form a hypothesis, simulate it, and then verify it under controlled load.</p>
  <p class="ko">그 시점에 선택지는 두 가지였다. 코드를 더 읽고, 더 많은 추측성 질문을 계속 던지거나. 아니면 이 문제를 전형적인 시스템 문제로 다루는 것이다. 가설을 세우고, 시뮬레이션으로 좁히고, 통제된 부하 실험으로 검증하는 방식이다.</p>
</div>

<div class="bilingual">
  <p class="en">I chose the second path. The admission gate was spread across several files, Redis state, Nakama presence streams, and the WebSocket lifecycle. The important question was no longer “what does this function do?” It was “in what order, and at what frequency, do these pieces interact under stress?”</p>
  <p class="ko">나는 두 번째 방법을 택했다. admission gate는 여러 파일에 걸쳐 있고, Redis 상태와 Nakama presence stream, WebSocket lifecycle까지 얽혀 있었다. 중요한 질문은 더 이상 “이 함수가 무엇을 하는가?”가 아니었다. “부하가 걸렸을 때 이 조각들이 어떤 순서와 빈도로 상호작용하는가?”였다.</p>
</div>

<div class="bilingual">
  <h2 class="en">Simulation ruled out the obvious theory</h2>
  <h2 class="ko">시뮬레이션은 가장 그럴듯한 가설부터 지워 나갔다</h2>
</div>

<div class="bilingual">
  <p class="en">My first hypothesis was feedback delay. Heartbeat-based session counting has delay built into it, so maybe that delay was causing oscillation. I wrote a Python simulator and pushed heartbeat intervals from 0.1 seconds to 10 seconds. Across 6,216 simulation runs, every case still converged. Delay existed, but delay was not the driver.</p>
  <p class="ko">첫 번째 가설은 피드백 지연이었다. heartbeat 기반 세션 카운트에는 애초에 지연이 있으니, 그 지연이 oscillation을 만든다고 의심했다. Python 시뮬레이터를 만들고 heartbeat 간격을 0.1초에서 10초까지 밀어봤다. 6,216회의 시뮬레이션 결과는 전부 수렴이었다. 지연은 존재했지만, 문제의 원인은 아니었다.</p>
</div>

<div class="bilingual">
  <p class="en">The more interesting hypothesis was ghost sessions. In the real server, WebSocket connections can stay alive even when the gate rejects the gameplay path. That means a client can be rejected and still keep pressure on the system. Once I modeled that behavior, the simulation began to show instability.</p>
  <p class="ko">더 유력했던 건 유령 세션 가설이었다. 실제 서버에서는 gate가 gameplay path를 거부해도 WebSocket 자체는 살아 있을 수 있다. 즉, 거부된 클라이언트가 시스템에 계속 압력을 주는 구조다. 이 동작을 시뮬레이션에 넣자 비로소 불안정성이 보이기 시작했다.</p>
</div>

<div class="bilingual">
  <p class="en">The counterintuitive part was the best clue. Faster cleanup was not always better. Shorter backoff was not always better. In the V4 simulation, a 1 second GC delay produced 34.7% divergence, 5 seconds gave 18.4%, and 30 seconds dropped to 0%. On the retry side, 0 second backoff produced 0% divergence, 1 second produced 37.3%, and 5 seconds fell to 4.1%.</p>
  <p class="ko">힌트가 되었던 건 오히려 직관에 반하는 결과였다. cleanup이 빠를수록 무조건 좋은 것도 아니었고, backoff가 짧을수록 무조건 좋은 것도 아니었다. V4 시뮬레이션에서 GC delay 1초는 발산률 34.7%, 5초는 18.4%, 30초는 0%였다. retry 쪽도 비슷했다. backoff 0초는 0%, 1초는 37.3%, 5초는 4.1%였다.</p>
</div>

<div class="bilingual">
  <p class="en">That kind of result almost never comes out of a code review. It comes out of a model.</p>
  <p class="ko">이런 종류의 인사이트는 코드 리뷰로는 거의 나오지 않는다. 모델을 돌려봐야 나온다.</p>
</div>

<div class="bilingual">
  <h2 class="en">Controlled load tests made the pattern look real</h2>
  <h2 class="ko">통제 실험은 패턴이 실제처럼 보이게 만들었다</h2>
</div>

<div class="bilingual">
  <p class="en">The simulation suggested that two strategies could flip places depending on load, so I built an automated experiment loop: switch strategy, flush Redis, restart the server, scrape Prometheus once per second, and collect the result. That let me run proper controlled tests instead of one-off anecdotes.</p>
  <p class="ko">시뮬레이션은 부하에 따라 두 전략의 우열이 뒤집힐 수 있다고 예측했다. 그래서 전략 전환, Redis flush, 서버 재시작, Prometheus 1초 스크래핑, 결과 수집까지 자동화한 실험 루프를 만들었다. 덕분에 “한 번 우연히 나온 결과”가 아니라 통제된 실험을 돌릴 수 있었다.</p>
</div>

<div class="table-wrap bilingual">
  <div class="en">
    <table>
      <thead>
        <tr>
          <th>CCU</th>
          <th>% of Cap</th>
          <th>B Errors</th>
          <th>A Errors</th>
          <th>Reading</th>
        </tr>
      </thead>
      <tbody>
        <tr>
          <td>1K</td>
          <td>20%</td>
          <td>3,256</td>
          <td>3,190</td>
          <td>roughly equal</td>
        </tr>
        <tr>
          <td>3K</td>
          <td>60%</td>
          <td>33,788</td>
          <td>22,237</td>
          <td>A wins</td>
        </tr>
        <tr>
          <td>5K</td>
          <td>100%</td>
          <td>28,549</td>
          <td>30,907</td>
          <td>B wins</td>
        </tr>
        <tr>
          <td>6K</td>
          <td>120%</td>
          <td>37,067</td>
          <td>43,172</td>
          <td>B wins</td>
        </tr>
        <tr>
          <td>8K</td>
          <td>160%</td>
          <td>14,736</td>
          <td>36,597</td>
          <td>B wins</td>
        </tr>
      </tbody>
    </table>
  </div>
  <div class="ko">
    <table>
      <thead>
        <tr>
          <th>CCU</th>
          <th>Cap 대비</th>
          <th>B Errors</th>
          <th>A Errors</th>
          <th>해석</th>
        </tr>
      </thead>
      <tbody>
        <tr>
          <td>1K</td>
          <td>20%</td>
          <td>3,256</td>
          <td>3,190</td>
          <td>거의 동등</td>
        </tr>
        <tr>
          <td>3K</td>
          <td>60%</td>
          <td>33,788</td>
          <td>22,237</td>
          <td>A 우세</td>
        </tr>
        <tr>
          <td>5K</td>
          <td>100%</td>
          <td>28,549</td>
          <td>30,907</td>
          <td>B 우세</td>
        </tr>
        <tr>
          <td>6K</td>
          <td>120%</td>
          <td>37,067</td>
          <td>43,172</td>
          <td>B 우세</td>
        </tr>
        <tr>
          <td>8K</td>
          <td>160%</td>
          <td>14,736</td>
          <td>36,597</td>
          <td>B 우세</td>
        </tr>
      </tbody>
    </table>
  </div>
</div>

<div class="bilingual">
  <p class="en">At 3K, A looked clearly better. At 5K and above, B took over. A later TTL sweep even suggested a neat interior optimum: 5 seconds at 3K, 15 seconds at 8K. For a while, this looked like a satisfying system story.</p>
  <p class="ko">3K에서는 A가 확실히 좋아 보였고, 5K 이상부터는 B가 우세해졌다. 뒤이어 돌린 TTL sweep에서는 더 그럴듯한 그림까지 나왔다. 3K에선 5초, 8K에선 15초가 최적처럼 보였다. 한동안은 꽤 예쁜 시스템 서사가 만들어진 것 같았다.</p>
</div>

<div class="bilingual">
  <h2 class="en">Then the result stopped reproducing</h2>
  <h2 class="ko">그런데 반복 횟수를 늘리자 결과가 사라졌다</h2>
</div>

<div class="bilingual">
  <p class="en">The turning point came when I increased the repeat count from 3 to 7. The pattern disappeared. The “best” TTL changed. The confidence intervals swallowed zero. The coefficient of variation on connection errors was 0.33, which meant that the same setup could vary wildly from run to run. At that point, the right question was no longer which strategy was better. It was why the experiment was so noisy in the first place.</p>
  <p class="ko">전환점은 반복 횟수를 3회에서 7회로 늘렸을 때 왔다. 패턴이 사라졌다. “최적”이라고 보였던 TTL도 뒤집혔고, confidence interval은 0을 포함했다. connection error의 run-to-run CV는 0.33이었다. 같은 조건에서도 결과가 크게 흔들린다는 뜻이다. 그 순간부터 중요한 질문은 “어느 전략이 더 좋은가?”가 아니었다. “왜 실험 자체가 이렇게 시끄러운가?”였다.</p>
</div>

<div class="bilingual">
  <p class="en">That was the point where the data stopped telling me to tune parameters and started telling me to suspect the code.</p>
  <p class="ko">그 시점부터 데이터는 파라미터를 더 만지라고 말하지 않았다. 코드를 의심하라고 말하고 있었다.</p>
</div>

<div class="bilingual">
  <h2 class="en">The first bug was over-admission</h2>
  <h2 class="ko">첫 번째 버그는 over-admission이었다</h2>
</div>

<div class="bilingual">
  <p class="en">When I went back to the code, the question had changed. I was no longer asking “what might go wrong?” I was asking “what would explain this amount of variance?” That is how the first bug finally became visible.</p>
  <p class="ko">코드를 다시 읽을 때 질문도 달라졌다. “무엇이 잘못될 수 있을까?”가 아니라 “이 정도 분산을 설명할 수 있는 경로가 무엇일까?”를 보기 시작했다. 그때 첫 번째 버그가 비로소 눈에 들어왔다.</p>
</div>

<div class="bilingual">
  <ol class="en">
    <li>A WebSocket connects and joins the stream.</li>
    <li><code>characters_enter</code> is rejected by the gate.</li>
    <li>Strategy A removes that session from the stream on reject.</li>
    <li>The same WebSocket retries.</li>
    <li>The gate sees space because that session is no longer counted.</li>
    <li>The session is admitted again without ever rejoining the stream.</li>
    <li>From there, the gate can keep admitting traffic it no longer sees.</li>
  </ol>
  <ol class="ko">
    <li>WebSocket이 연결되면서 stream에 들어간다.</li>
    <li><code>characters_enter</code> RPC가 gate에서 거부된다.</li>
    <li>Strategy A는 거부 시 그 session을 stream에서 제거한다.</li>
    <li>같은 WebSocket이 다시 재시도한다.</li>
    <li>gate는 그 session을 더 이상 세지 않으므로 “자리가 있다”고 판단한다.</li>
    <li>session은 stream에 다시 join하지 않은 채 재입장한다.</li>
    <li>그 순간부터 gate는 자신이 보지 못하는 트래픽을 계속 admit할 수 있다.</li>
  </ol>
</div>

<div class="bilingual">
  <p class="en">This was a bug that static analysis could find, but only after the experiment gave me the right question. Once I suspected over-admission, the data lined up perfectly: at 8K, Strategy A had abnormal error counts, the gate was almost always open, and admits per second were dramatically higher than Strategy B.</p>
  <p class="ko">이 버그는 정적 분석으로도 찾을 수 있는 종류였다. 다만 실험이 먼저 올바른 질문을 만들어줬을 때만 보였다. over-admission을 의심하고 나니 데이터가 딱 맞아떨어졌다. 8K에서 Strategy A는 비정상적으로 높은 error를 보였고, gate는 거의 항상 열려 있었으며, admits/sec도 Strategy B보다 훨씬 높았다.</p>
</div>

<div class="bilingual">
  <h2 class="en">The second bug only showed up in a live path</h2>
  <h2 class="ko">두 번째 버그는 실제 경로에서만 드러났다</h2>
</div>

<div class="bilingual">
  <p class="en">After fixing over-admission, I implemented an admitted-count mode with <code>QUEUE_SESSION_COUNT_SOURCE=admitted</code> and reran the experiment. Strategy A immediately failed across the board.</p>
  <p class="ko">over-admission을 수정한 뒤, <code>QUEUE_SESSION_COUNT_SOURCE=admitted</code> 모드를 구현하고 실험을 다시 돌렸다. Strategy A는 즉시 전 구간에서 실패했다.</p>
</div>

<div class="bilingual">
  <blockquote class="en">session_id missing for admitted gate</blockquote>
  <blockquote class="ko">session_id missing for admitted gate</blockquote>
</div>

<div class="bilingual">
  <p class="en">That message was repeated over and over in the server logs. The issue turned out to be simple and nasty: Nakama does not populate <code>RUNTIME_CTX_SESSION_ID</code> for the HTTP RPC path. It exists for WebSocket sessions, but my load client was calling the RPC through HTTP POST.</p>
  <p class="ko">서버 로그에는 이 메시지가 반복해서 찍히고 있었다. 원인은 단순하지만 까다로웠다. Nakama는 HTTP RPC 경로에서는 <code>RUNTIME_CTX_SESSION_ID</code>를 채워주지 않는다. WebSocket session 경로에서만 들어오는 값인데, 내가 만든 부하 클라이언트는 RPC를 HTTP POST로 호출하고 있었다.</p>
</div>

<div class="bilingual">
  <p class="en">This bug is exactly the sort of thing static analysis tends to miss. Unit tests still passed because the test harness injected a fake session id with <code>context.WithValue</code>. The failure only existed on the real framework path under real load. I could have found it by reading Nakama internals, but the load logs found it in minutes.</p>
  <p class="ko">이건 정적 분석이 놓치기 쉬운 전형적인 버그였다. 단위 테스트는 여전히 통과했다. 테스트 하네스가 <code>context.WithValue</code>로 가짜 session id를 주입하고 있었기 때문이다. 실패는 실제 프레임워크 경로, 그것도 실제 부하 상황에서만 나타났다. Nakama 내부 코드를 끝까지 추적하면 찾을 수도 있었겠지만, 부하 테스트 로그는 몇 분 만에 원인을 보여줬다.</p>
</div>

<div class="bilingual">
  <h2 class="en">After both fixes, the A/B story disappeared</h2>
  <h2 class="ko">두 버그를 수정하고 나니 A/B 서사 자체가 사라졌다</h2>
</div>

<div class="bilingual">
  <p class="en">After fixing both bugs, I ran 28 more controlled tests. The earlier crossover story was gone.</p>
  <p class="ko">두 버그를 모두 수정한 뒤 28회의 통제 실험을 다시 돌렸다. 이전에 보였던 전략 간 crossover 이야기는 사라졌다.</p>
</div>

<div class="table-wrap bilingual">
  <div class="en">
    <table>
      <thead>
        <tr>
          <th>CCU</th>
          <th>A Errors</th>
          <th>B Errors</th>
          <th>Delta (A-B)</th>
          <th>95% CI</th>
          <th>Significant?</th>
        </tr>
      </thead>
      <tbody>
        <tr>
          <td>3K</td>
          <td>2,037</td>
          <td>2,011</td>
          <td>+27</td>
          <td>[-466, +421]</td>
          <td>No</td>
        </tr>
        <tr>
          <td>8K</td>
          <td>7,352</td>
          <td>8,336</td>
          <td>-984</td>
          <td>[-4,132, +2,350]</td>
          <td>No</td>
        </tr>
      </tbody>
    </table>
  </div>
  <div class="ko">
    <table>
      <thead>
        <tr>
          <th>CCU</th>
          <th>A Errors</th>
          <th>B Errors</th>
          <th>Delta (A-B)</th>
          <th>95% CI</th>
          <th>유의?</th>
        </tr>
      </thead>
      <tbody>
        <tr>
          <td>3K</td>
          <td>2,037</td>
          <td>2,011</td>
          <td>+27</td>
          <td>[-466, +421]</td>
          <td>No</td>
        </tr>
        <tr>
          <td>8K</td>
          <td>7,352</td>
          <td>8,336</td>
          <td>-984</td>
          <td>[-4,132, +2,350]</td>
          <td>No</td>
        </tr>
      </tbody>
    </table>
  </div>
</div>

<div class="bilingual">
  <p class="en">That was the real answer. Strategy A was not secretly better at one load level and worse at another. The earlier pattern had mostly been the shape of a bug. Once the bug was gone, the strategy gap mostly disappeared with it.</p>
  <p class="ko">그게 진짜 답이었다. Strategy A가 어떤 구간에서는 더 좋고 다른 구간에서는 더 나쁜 것이 아니었다. 이전에 보였던 패턴 대부분은 버그가 만들어낸 그림자였다. 버그가 사라지자 전략 차이도 거의 함께 사라졌다.</p>
</div>

<div class="bilingual">
  <p class="en">One more runtime truth showed up here as well. Strategy B closed the gate much more often, but final connection errors did not improve much. Rejected clients did not vanish. They piled up, waited, and then rushed back in together when the gate reopened. Static analysis does not warn you about thundering-herd dynamics. The runtime does.</p>
  <p class="ko">여기서 또 하나의 런타임 진실이 보였다. Strategy B는 gate를 훨씬 자주 닫았지만 최종 connection error는 크게 개선되지 않았다. 거부된 클라이언트가 사라지는 게 아니라 backlog에 쌓였다가, gate가 다시 열릴 때 한꺼번에 밀려 들어왔기 때문이다. 정적 분석은 이런 thundering herd 동역학을 경고해주지 않는다. 런타임이 알려준다.</p>
</div>

<div class="bilingual">
  <h2 class="en">What AI was good at, and what it was not</h2>
  <h2 class="ko">이번 일로 다시 확인한 AI의 역할</h2>
</div>

<div class="bilingual">
  <p class="en">AI was still useful throughout the whole process. It helped me write the server code quickly. It helped me refactor. And once I had narrowed the question far enough, it could have helped me confirm bug 1 almost immediately.</p>
  <p class="ko">이번 과정에서도 AI는 충분히 유용했다. 서버 코드를 빠르게 짜는 데 도움이 되었고, 리팩터링에도 도움이 되었고, 질문이 충분히 좁혀진 뒤에는 버그 1 같은 문제를 거의 즉시 확인하는 데도 쓸 수 있었다.</p>
</div>

<div class="bilingual">
  <p class="en">What it could not do on its own was tell me how a live system would behave under pressure. It could not see emergent behavior from code alone. It could not know that a framework-specific HTTP context would drop a value that my tests were injecting. It could not tell me that my “interesting result” was actually noise unless I gave it the statistical framing first.</p>
  <p class="ko">하지만 AI 혼자서는 부하가 걸린 실시간 시스템이 어떻게 반응할지 말해주지 못했다. 코드만 보고 emergent behavior를 읽어내지 못했고, 테스트에서는 주입되던 값이 프레임워크의 실제 HTTP context에서는 빠진다는 사실도 스스로 알 수 없었다. 통계적 맥락을 내가 먼저 만들어주기 전에는 “흥미로운 결과”가 사실 노이즈라는 판단도 해주지 못했다.</p>
</div>

<div class="bilingual">
  <p class="en">That is the line I now draw more clearly. Static analysis answers local questions. Dynamic analysis answers system questions. Good debugging work needs both.</p>
  <p class="ko">이 경험 이후로 나는 경계를 더 분명하게 보게 됐다. 정적 분석은 로컬한 질문에 답한다. 동적 분석은 시스템 수준의 질문에 답한다. 좋은 디버깅에는 둘 다 필요하다.</p>
</div>

<div class="bilingual">
  <h2 class="en">Automation was what made this practical</h2>
  <h2 class="ko">그리고 이 사이클을 가능하게 한 건 자동화였다</h2>
</div>

<div class="bilingual">
  <p class="en">Without automation, I probably would have stopped much earlier and convinced myself with a cleaner story than the data deserved. The 58-run controlled experiment took around five hours. The 72-run TTL sweep took around six. The 28-run retest after the fixes took about two and a half hours. Manually, that would have stretched across days.</p>
  <p class="ko">자동화가 없었다면 나는 훨씬 일찍 멈췄을 것이다. 그리고 데이터가 허락하지도 않는 깔끔한 서사를 스스로 믿었을 가능성이 크다. 58회 통제 실험은 약 5시간, 72회 TTL sweep은 약 6시간, 수정 후 28회 재실험은 약 2시간 30분이 걸렸다. 이걸 수동으로 했다면 며칠짜리 작업이 된다.</p>
</div>

<div class="bilingual">
  <p class="en">That changes the engineering culture around debugging. If rerunning a serious experiment takes two days, you guess. If it takes a couple of hours, you rerun it. AI automated a large part of the coding work. The load pipeline automated a large part of the experimental work. I needed both automations to close the loop.</p>
  <p class="ko">이 차이는 디버깅 문화 자체를 바꾼다. 진짜 실험을 다시 돌리는 데 이틀이 걸리면 사람은 추정하게 된다. 몇 시간이면 끝난다면 다시 돌려서 확인한다. AI는 코딩 작업의 큰 부분을 자동화했고, 부하 실험 파이프라인은 실험 작업의 큰 부분을 자동화했다. 이 루프를 닫으려면 두 자동화가 모두 필요했다.</p>
</div>

<div class="bilingual">
  <h2 class="en">Would this have blown up in production?</h2>
  <h2 class="ko">이 버그가 실제 운영에서 터졌을까?</h2>
</div>

<div class="bilingual">
  <p class="en">Not under the default setup. That part is important. Both bugs were tied to alternative strategies, not the baseline configuration. The real risk was optimization. If I had looked at the code, decided admitted counting was more correct, and turned it on before a high-load event, I could have shipped a failure path that looked rational on paper and collapsed in practice.</p>
  <p class="ko">기본 설정 그대로였다면 바로 터지지는 않았을 것이다. 이 점은 중요하다. 두 버그 모두 기본 설정이 아니라 대안 전략을 켰을 때 발현되는 문제였다. 진짜 위험은 “최적화”였다. 코드를 읽고 admitted counting이 더 정확해 보인다고 판단한 뒤, 고부하 이벤트 전에 그 설정을 켰다면, 종이 위에서는 합리적으로 보이지만 실제로는 망가지는 경로를 그대로 운영에 넣었을 가능성이 높다.</p>
</div>

<div class="bilingual">
  <p class="en">That is the practical value of experiment-driven debugging. It catches the bugs you introduce while trying to be clever.</p>
  <p class="ko">실험 기반 디버깅의 실질적인 가치는 바로 여기에 있다. 더 잘해보겠다고 손댄 최적화가 실제로는 장애 경로인지, 운영 전에 걸러낼 수 있다는 점이다.</p>
</div>

<div class="bilingual">
  <h2 class="en">The loop I trust now</h2>
  <h2 class="ko">지금 내가 더 믿게 된 디버깅 루프</h2>
</div>

<div class="bilingual">
  <p class="en">If I had relied on code review alone, I probably would have walked away with a plausible explanation and the wrong fix. What finally worked was a loop: run experiments until the system gives me a concrete question, read the code with that question in mind, then go back to the running system and verify the answer.</p>
  <p class="ko">코드 리뷰만 믿고 갔다면, 아마 그럴듯한 설명과 틀린 수정안을 들고 끝냈을 것이다. 실제로 효과가 있었던 건 다른 루프였다. 실험을 돌려서 시스템이 먼저 구체적인 질문을 만들게 하고, 그 질문을 들고 코드를 다시 읽고, 마지막으로 돌아가서 실제 시스템에서 답이 맞는지 확인하는 방식이다.</p>
</div>

<div class="bilingual">
  <p class="en">AI made the coding side faster. Automation made the experimental side faster. Neither one replaced the other. In this debugging session, static analysis did not lose to dynamic analysis. Static analysis needed dynamic analysis to know where to look.</p>
  <p class="ko">AI는 코딩 쪽을 빠르게 만들었고, 자동화는 실험 쪽을 빠르게 만들었다. 둘 중 하나가 다른 하나를 대체한 게 아니었다. 이번 디버깅에서 정적 분석이 동적 분석에게 “진 것”도 아니다. 정적 분석은 어디를 봐야 할지 알려주는 동적 분석이 필요했다.</p>
</div>
