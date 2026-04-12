---
title: "AI Wrote the Code, but Could Not Find the Bug"
title_kr: "AI가 코드는 짰지만, 버그는 찾지 못했다"
date: 2026-04-03 12:00:00 +0800
categories:
  - server-engineering
excerpt_en: "AI helped me build and review an MMORPG server, but the real bugs only surfaced after 6,216 simulation runs and more than 100 load tests."
excerpt_kr: "AI는 MMORPG 서버를 만드는 데 큰 도움이 됐지만, 진짜 버그는 6,216회의 시뮬레이션과 100회가 넘는 부하 실험 끝에야 드러났다."
---

<div class="bilingual">
  <p class="en lede">While building a large online game server, I thought the code itself was mostly fine. Types passed, tests passed, and individual functions looked clean enough. The problems that surfaced later were not really line-level code bugs. They were closer to design bugs, the kind that only show up in chaos tests or end-to-end live-service simulations.</p>
  <p class="ko lede">대규모 온라인 게임 서버를 만드는 과정에서, 나는 코드 자체는 꽤 괜찮다고 생각하며 만들었다. 타입도 맞고 테스트도 통과했고, 개별 함수만 놓고 보면 큰 문제는 없어 보였다. 그런데 나중에 드러난 문제는 코드 한 줄의 오류라기보다 설계와 런타임 상호작용에서 생기는 종류의 것이었다. 이런 문제는 대개 카오스 테스트나 E2E 라이브 서비스 시뮬레이션에서 드러났다.</p>
</div>

<div class="bilingual">
  <p class="en">The first few experiments only gave me a vague sense that something was off. After thousands of simulation runs and more than 100 load tests, that stopped looking like a coincidence. Something in the design was clearly wrong. Ordinary code review could not diagnose it, and even asking AI to reread the whole codebase did not get me there. I only managed to expose the problem after 6,216 simulation runs and more than 100 controlled load experiments.</p>
  <p class="ko">처음 몇 번의 실험에서는 그냥 뭔가 이상하다는 느낌 정도였다. 하지만 시뮬레이션을 수천 번 반복하고 실제 부하 실험을 100회 넘게 돌리면서, 상식적으로 설명되지 않는 현상이 분명해졌다. 어딘가 설계가 잘못됐다는 감은 있었지만, 일반적인 코드 리뷰로도, AI에게 코드베이스 전체를 다시 읽혀봐도 원인을 잡아내지 못했다. 결국 6,216회의 시뮬레이션과 100회가 넘는 통제된 부하 실험 끝에야 문제를 드러낼 수 있었다.</p>
</div>

<div class="bilingual">
  <h2 class="en">AI was still excellent at the local work</h2>
  <h2 class="ko">그래도 AI는 로컬한 코드 작업에는 정말 강했다</h2>
</div>

<div class="bilingual">
  <p class="en">For context, I am building an MMORPG server on top of Nakama, with Go plugins carrying the game-specific logic. Most of the implementation work goes through tools like Claude Code and Codex CLI. They are very good at writing code, reviewing it, and cleaning it up.</p>
  <p class="ko">배경을 간단히 말하면, 나는 지금 Nakama 위에 Go 플러그인을 얹는 구조로 MMORPG 서버를 만들고 있다. 구현 과정의 대부분은 Claude Code나 Codex CLI 같은 도구와 함께 간다. 코드를 쓰고, 리뷰하고, 정리하는 데 정말 큰 도움이 된다.</p>
</div>

<div class="bilingual">
  <p class="en">On local questions, AI is excellent. It passes type checks, gets tests green, and catches small edge cases surprisingly well. If I had to compare it to a human team, one AI session feels like a competent senior driving the task with several fast juniors implementing around it. The problem is that local correctness is not the same thing as system behavior.</p>
  <p class="ko">로컬한 질문에 대해서는 AI가 정말 강하다. 타입 체크도 통과하고, 테스트도 통과하고, 자잘한 엣지 케이스도 잘 잡아낸다. 사람 팀에 비유하면, 한 개의 AI 세션이 꽤 괜찮은 시니어 한 명이 방향을 잡고 손 빠른 주니어 몇 명이 주변 구현을 붙여주는 느낌에 가깝다. 문제는 로컬한 정확성이 곧 시스템 동작을 뜻하지는 않는다는 점이다.</p>
</div>

<div class="bilingual">
  <p class="en">A server can look perfectly clean in a diff and still fail in ugly ways once Redis state, presence streams, WebSocket lifecycles, retries, and sudden load spikes start interacting at the same time. That was the gap I ran into.</p>
  <p class="ko">diff 상으로는 깔끔해 보여도 Redis 상태, presence stream, WebSocket lifecycle, retry, 순간적인 부하 급증이 동시에 얽히기 시작하면 서버는 아주 지저분한 방식으로 무너질 수 있다. 내가 이번에 부딪힌 문제가 정확히 그런 종류였다.</p>
</div>

<div class="bilingual">
  <h2 class="en">The first signal was upside down</h2>
  <h2 class="ko">첫 징후는 직관과 정반대였다</h2>
</div>

<div class="bilingual">
  <p class="en">The admission gate on the server was behaving backward. If I raised the session cap per node, connection errors were supposed to go down. Instead, they exploded.</p>
  <p class="ko">서버의 admission gate가 이상하게 움직이고 있었다. 원래라면 노드당 세션 cap을 올리면 connection error가 줄어야 맞다. 그런데 실제로는 반대로 폭증했다.</p>
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
          <td>Cap up 50%, errors up about 3x</td>
        </tr>
        <tr>
          <td>16,000</td>
          <td>+4,185%</td>
          <td>Cap up 2x, errors up about 42x</td>
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
          <td>cap 50% 증가, error는 약 3배</td>
        </tr>
        <tr>
          <td>16,000</td>
          <td>+4,185%</td>
          <td>cap 2배 증가, error는 약 42배</td>
        </tr>
      </tbody>
    </table>
  </div>
</div>

<div class="bilingual">
  <p class="en">I asked AI to reread the admission gate and explain why higher caps could lead to more errors. It came back with plausible suspects: stale heartbeat counts, Redis races, garbage collection timing. All of them sounded reasonable. None of them were the real cause.</p>
  <p class="ko">나는 AI에게 admission gate 코드를 다시 읽히고, 왜 cap을 올렸는데 error가 늘 수 있는지 물었다. stale heartbeat count, Redis race, garbage collection timing 같은 그럴듯한 후보들이 돌아왔다. 다 말은 됐다. 하지만 진짜 원인은 아니었다.</p>
</div>

<div class="bilingual">
  <p class="en">That was the moment I had to admit something simple. Static analysis can tell me whether code is locally coherent. It cannot reliably tell me what happens when eight thousand sessions begin retrying, drifting, and colliding inside a real runtime.</p>
  <p class="ko">그 시점에서 아주 단순한 사실을 인정해야 했다. 정적 분석은 코드가 로컬하게 일관적인지는 말해줄 수 있다. 하지만 8,000개의 세션이 실제 런타임에서 동시에 retry하고, 엇갈리고, 서로 충돌하기 시작할 때 무슨 일이 벌어지는지는 믿을 만하게 말해주지 못한다.</p>
</div>

<div class="bilingual">
  <h2 class="en">I stopped arguing with the code and started running experiments</h2>
  <h2 class="ko">코드만 붙잡지 않고 실험을 돌리기 시작했다</h2>
</div>

<div class="bilingual">
  <p class="en">At that point I had two options. I could keep reading code and keep asking speculative questions. Or I could treat this as a systems problem: form a hypothesis, simulate it, and then verify it under controlled load. I chose the second path.</p>
  <p class="ko">그 시점의 선택지는 사실 두 가지였다. 코드를 더 읽으면서 추측성 질문을 계속 던지거나, 아니면 이 문제를 전형적인 시스템 문제로 다루는 것이다. 가설을 세우고, 시뮬레이션으로 좁히고, 통제된 부하 실험으로 검증하는 방식이다. 나는 후자를 택했다.</p>
</div>

<div class="bilingual">
  <p class="en">In another environment, this kind of debugging would have been expensive. But I was still in development, and I had a very cheap, very fast team available: AI could write the harnesses and automation while I focused on the questions. At this stage, the important question was no longer <em>what does this function do?</em> It was <em>in what order, and how often, do these pieces interact under pressure?</em></p>
  <p class="ko">예전 같았으면 이런 디버깅은 꽤 비싼 작업이었다. 하지만 아직 개발 중이었고, 아주 싸고 빠른 팀이 있었다. AI에게 하네스와 자동화 코드를 맡기고, 나는 질문을 좁히는 데 집중할 수 있었다. 이 단계에서 중요한 질문은 더 이상 “이 함수가 무엇을 하는가?”가 아니었다. “부하가 걸렸을 때 이 조각들이 어떤 순서와 빈도로 서로 상호작용하는가?”가 핵심이었다.</p>
</div>

<div class="bilingual">
  <h2 class="en">Simulation ruled out the obvious theory</h2>
  <h2 class="ko">시뮬레이션은 그럴듯한 가설부터 지워나갔다</h2>
</div>

<div class="bilingual">
  <p class="en">My first hypothesis was feedback delay. Heartbeat-based session counting already has delay built in, so maybe that delay was creating oscillation. I wrote a Python simulator and pushed the heartbeat interval from 0.1 seconds to 10 seconds. Across 6,216 simulation runs, every case still converged. Delay existed, but delay was not the driver.</p>
  <p class="ko">첫 번째 가설은 피드백 지연이었다. heartbeat 기반 세션 카운트에는 원래 지연이 있으니, 그 지연이 oscillation을 만들고 있을 거라고 의심했다. 그래서 Python 시뮬레이터를 만들고 heartbeat 간격을 0.1초에서 10초까지 밀어봤다. 6,216회의 시뮬레이션 결과는 전부 수렴이었다. 지연은 있었지만, 핵심 원인은 아니었다.</p>
</div>

<div class="bilingual">
  <p class="en">The next hypothesis was more interesting: ghost sessions. On the real server, the gate could reject the gameplay path while the WebSocket itself stayed alive. That meant rejected clients could keep pushing on the system. Once I modeled that behavior, instability finally started to show up.</p>
  <p class="ko">그다음으로 유력했던 건 유령 세션 가설이었다. 실제 서버에서는 gate가 gameplay path를 거부해도 WebSocket 자체는 살아 있을 수 있다. 즉, 거부된 클라이언트가 시스템에 계속 압력을 주는 구조다. 이 동작을 시뮬레이션에 넣고 나서야 비로소 불안정성이 눈에 띄기 시작했다.</p>
</div>

<div class="bilingual">
  <p class="en">The best clue was how non-intuitive the result looked. Faster cleanup was not always better. Shorter backoff was not always better. In the V4 simulation, a 1-second GC delay produced 34.7% divergence, 5 seconds produced 18.4%, and 30 seconds produced 0%. On retries, 0-second backoff gave 0%, 1 second gave 37.3%, and 5 seconds gave 4.1%.</p>
  <p class="ko">힌트가 되었던 건 오히려 결과가 직관을 배신했다는 점이다. cleanup이 빠를수록 무조건 좋은 것도 아니었고, backoff가 짧을수록 무조건 좋은 것도 아니었다. V4 시뮬레이션에서 GC delay 1초는 발산률 34.7%, 5초는 18.4%, 30초는 0%였다. retry도 비슷했다. backoff 0초는 0%, 1초는 37.3%, 5초는 4.1%였다.</p>
</div>

<div class="bilingual">
  <p class="en">You almost never get that kind of insight from a code review. You get it from a model.</p>
  <p class="ko">이런 인사이트는 코드 리뷰만으로는 거의 나오지 않는다. 모델을 돌려봐야 나온다.</p>
</div>

<div class="bilingual">
  <h2 class="en">Controlled load tests made the pattern look real</h2>
  <h2 class="ko">통제된 부하 실험은 패턴을 실제처럼 보이게 만들었다</h2>
</div>

<div class="bilingual">
  <p class="en">The simulation suggested that two strategies might trade places depending on load, so I built an automated experiment loop: switch strategy, flush Redis, restart the server, scrape Prometheus every second, and collect the result. That let me run proper controlled tests instead of building stories out of one-off anecdotes.</p>
  <p class="ko">시뮬레이션은 부하에 따라 두 전략의 우열이 뒤집힐 수 있다고 예측했다. 그래서 전략 전환, Redis flush, 서버 재시작, Prometheus 1초 스크래핑, 결과 수집까지 전부 자동화한 실험 루프를 만들었다. 덕분에 “한 번 우연히 나온 결과”가 아니라, 조건을 통제한 실험을 반복할 수 있었다.</p>
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
  <p class="en">At 3K, A looked clearly better. From 5K upward, B started winning. It was a very clean story. Then TTL sweeps made it look even cleaner: 5 seconds seemed best at 3K, 15 seconds seemed best at 8K. For a while, it felt like a neat systems result.</p>
  <p class="ko">3K에서는 A가 확실히 좋아 보였고, 5K를 넘기면서부터는 B가 우세해졌다. 겉으로만 보면 굉장히 그럴듯한 그림이었다. 이어서 TTL sweep을 돌려보니 3K에서는 5초, 8K에서는 15초가 최적처럼 보였다. 한동안은 꽤 그럴듯한 시스템적 발견처럼 읽혔다.</p>
</div>

<div class="bilingual">
  <p class="en">Meanwhile, Claude Code and Codex CLI reacted to the weirdness as if it were a major discovery. That is something worth watching. Once enough context accumulates, AI can become overly eager to build a compelling narrative around noisy data.</p>
  <p class="ko">그 와중에 Claude Code와 Codex CLI는 이런 특이한 패턴을 아주 큰 발견처럼 받아들였다. 이 부분은 꽤 조심해서 봐야 한다. 맥락이 쌓이면 AI는 노이즈 위에도 그럴듯한 서사를 과하게 만들어내는 경향이 있다.</p>
</div>

<div class="bilingual">
  <h2 class="en">When I increased the repetitions, the story collapsed</h2>
  <h2 class="ko">반복 횟수를 늘리자 서사가 무너졌다</h2>
</div>

<div class="bilingual">
  <p class="en">I did not lock in the conclusion. I increased the repetition count. One batch took about five hours, but it was automated, so I could just let it run. The turning point came when I increased repeats from 3 to 7. The “big discovery” disappeared.</p>
  <p class="ko">나는 여기서 결론을 확정하지 않았다. 반복 횟수를 늘렸다. 한 세트에 약 5시간이 걸렸지만 자동화가 되어 있었기 때문에 그냥 돌려두면 됐다. 전환점은 반복 횟수를 3회에서 7회로 늘렸을 때 왔다. AI가 호들갑을 떨던 “대발견”이 사라진 것이다.</p>
</div>

<div class="bilingual">
  <p class="en">The pattern fell apart. The TTL that had looked optimal flipped. The confidence interval crossed zero. Run-to-run CV on connection errors was 0.33. Same condition, very different outcomes. From that point on, the important question was no longer <em>which strategy is better?</em> It was <em>why is the experiment itself this noisy?</em></p>
  <p class="ko">패턴이 무너졌다. 최적처럼 보였던 TTL은 뒤집혔고, confidence interval은 0을 포함했다. connection error의 run-to-run CV는 0.33이었다. 같은 조건에서도 결과가 크게 흔들린다는 뜻이다. 그 순간부터 중요한 질문은 “어느 전략이 더 좋은가?”가 아니었다. “왜 실험 자체가 이렇게 시끄러운가?”가 핵심이 됐다.</p>
</div>

<div class="bilingual">
  <p class="en">This was where AI and I diverged. AI wanted more hypotheses and more experiments. I thought it was time to suspect the code.</p>
  <p class="ko">여기서부터 AI와 인간의 해석이 갈렸다. AI는 더 많은 가설과 추가 실험으로 밀어붙이려 했고, 나는 오히려 파라미터를 더 만지기 전에 코드를 의심해야 한다고 생각했다.</p>
</div>

<div class="bilingual">
  <h2 class="en">The first bug was over-admission</h2>
  <h2 class="ko">첫 번째 버그는 over-admission이었다</h2>
</div>

<div class="bilingual">
  <p class="en">When I reread the code, the question had changed. I was no longer asking <em>what can go wrong?</em> I was asking <em>what path could explain this much variance?</em> That was when the first bug finally came into view.</p>
  <p class="ko">코드를 다시 읽을 때 질문도 달라졌다. “무엇이 잘못될 수 있을까?”가 아니라 “이 정도 분산을 설명할 수 있는 경로가 어디인가?”를 보기 시작했다. 그때 비로소 첫 번째 버그가 눈에 들어왔다.</p>
</div>

<div class="bilingual">
  <div class="en">
    <ol>
      <li>A WebSocket connects and joins the stream.</li>
      <li>The <code>characters_enter</code> RPC gets rejected by the gate.</li>
      <li>Strategy A removes that session from the stream on rejection.</li>
      <li>The same WebSocket retries.</li>
      <li>The gate no longer counts that session, so it believes capacity exists and readmits traffic it can no longer see.</li>
    </ol>
  </div>
  <div class="ko">
    <ol>
      <li>WebSocket이 연결되면서 stream에 들어간다.</li>
      <li><code>characters_enter</code> RPC가 gate에서 거부된다.</li>
      <li>Strategy A는 거부 시 그 session을 stream에서 제거한다.</li>
      <li>같은 WebSocket이 다시 재시도한다.</li>
      <li>gate는 그 session을 더 이상 세지 않으므로 자리가 있다고 판단하고, 자신이 보지 못하는 트래픽을 계속 admit하게 된다.</li>
    </ol>
  </div>
</div>

<div class="bilingual">
  <p class="en">This was a bug that static analysis could have found. The catch was that it only became visible after the experiments told me what kind of question to ask. Once I suspected over-admission, the data snapped into place. At 8K, Strategy A showed abnormally high errors, the gate stayed open almost all the time, and admits per second were far higher than Strategy B.</p>
  <p class="ko">이 버그는 정적 분석으로도 찾을 수 있는 종류였다. 다만 아무 때나 보이는 버그는 아니었다. 실험이 먼저 “어떤 질문을 해야 하는가”를 만들어줬을 때만 보이는 종류였다. over-admission을 의심하고 나니 데이터가 딱 맞아떨어졌다. 8K에서 Strategy A는 비정상적으로 높은 error를 보였고, gate는 거의 항상 열려 있었으며, admits/sec도 Strategy B보다 훨씬 높았다.</p>
</div>

<div class="bilingual">
  <h2 class="en">The second bug only appeared on the real path</h2>
  <h2 class="ko">두 번째 버그는 실제 경로에서만 드러났다</h2>
</div>

<div class="bilingual">
  <p class="en">After fixing over-admission, I implemented <code>QUEUE_SESSION_COUNT_SOURCE=admitted</code> and reran the experiments. Strategy A immediately failed everywhere. The server logs kept printing <code>session_id missing for admitted gate</code>.</p>
  <p class="ko">over-admission을 수정한 뒤 <code>QUEUE_SESSION_COUNT_SOURCE=admitted</code> 모드를 구현하고 실험을 다시 돌렸더니, Strategy A는 전 구간에서 바로 실패했다. 서버 로그에는 <code>session_id missing for admitted gate</code>가 계속 찍혔다.</p>
</div>

<div class="bilingual">
  <p class="en">The cause was simple but subtle. Nakama does not populate <code>RUNTIME_CTX_SESSION_ID</code> on the HTTP RPC path. That value only exists on the WebSocket session path, and my load client was calling the RPC through HTTP POST.</p>
  <p class="ko">원인은 단순하지만 까다로웠다. Nakama는 HTTP RPC 경로에서는 <code>RUNTIME_CTX_SESSION_ID</code>를 채워주지 않는다. 이 값은 WebSocket session 경로에서만 들어오는데, 내가 만든 부하 클라이언트는 RPC를 HTTP POST로 호출하고 있었다.</p>
</div>

<div class="bilingual">
  <p class="en">This is exactly the kind of bug static analysis misses easily. Unit tests still passed because the harness injected a fake session id with <code>context.WithValue</code>. The failure only appeared on the real framework path, under real load. I could probably have found it by tracing Nakama internals, but load test logs showed the cause in minutes.</p>
  <p class="ko">이건 정적 분석이 놓치기 쉬운 전형적인 버그였다. 단위 테스트는 계속 통과했다. 테스트 하네스가 <code>context.WithValue</code>로 가짜 session id를 주입하고 있었기 때문이다. 실패는 실제 프레임워크 경로, 그것도 실제 부하 상황에서만 나타났다. Nakama 내부 코드를 끝까지 추적하면 찾을 수도 있었겠지만, 부하 테스트 로그는 몇 분 만에 원인을 보여줬다.</p>
</div>

<div class="bilingual">
  <h2 class="en">Once the bugs were gone, the A/B story disappeared too</h2>
  <h2 class="ko">버그가 사라지자 A/B 서사도 함께 사라졌다</h2>
</div>

<div class="bilingual">
  <p class="en">After fixing both bugs, I reran 28 controlled experiments. The crossover story between the two strategies vanished.</p>
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
  <p class="en">The conclusion turned out to be much simpler than the earlier story. Strategy A was not better in one region and worse in another. Most of the earlier pattern was just a shadow cast by the bugs. Once the bugs disappeared, most of the strategy difference disappeared too.</p>
  <p class="ko">결론은 훨씬 단순했다. Strategy A가 어떤 구간에서는 더 좋고, 다른 구간에서는 더 나쁜 것이 아니었다. 처음에 보였던 패턴 대부분은 버그가 만든 그림자였다. 버그가 사라지자 전략 차이도 같이 흐려졌다.</p>
</div>

<div class="bilingual">
  <p class="en">One more runtime truth showed up here. Strategy B closed the gate much more often, but final connection errors barely improved. Rejected clients did not vanish. They piled up in the backlog and rushed in together when the gate reopened. Static analysis does not warn you about thundering herd dynamics. The runtime does.</p>
  <p class="ko">여기서 또 하나의 런타임 진실이 드러났다. Strategy B는 gate를 훨씬 자주 닫았지만 최종 connection error는 크게 개선되지 않았다. 거부된 클라이언트가 사라지는 게 아니라 backlog에 쌓였다가, gate가 다시 열릴 때 한꺼번에 밀려 들어왔기 때문이다. 이런 thundering herd 동역학은 정적 분석만으로는 잘 보이지 않는다. 런타임을 봐야 알 수 있다.</p>
</div>

<div class="bilingual">
  <h2 class="en">What this changed about how I use AI</h2>
  <h2 class="ko">이번 일로 AI를 쓰는 방식도 다시 정리됐다</h2>
</div>

<div class="bilingual">
  <p class="en">AI was still very useful throughout the whole process. It helped me write the server, refactor it, and once the question was narrow enough, confirm bugs like the first one very quickly. But AI alone could not tell me how a real-time system under load would behave. It could not infer emergent behavior just from reading the code, and it could not know that a value injected in tests was absent on the framework's real HTTP path.</p>
  <p class="ko">이번 과정에서도 AI는 충분히 유용했다. 서버 코드를 빠르게 짜는 데 도움이 되었고, 리팩터링에도 도움이 되었고, 질문이 충분히 좁혀진 뒤에는 첫 번째 버그 같은 문제를 거의 즉시 확인하는 데도 쓸 수 있었다. 하지만 AI 혼자서는 부하가 걸린 실시간 시스템이 어떻게 반응할지 말해주지 못했다. 코드만 보고 emergent behavior를 읽어내지 못했고, 테스트에서는 주입되던 값이 프레임워크의 실제 HTTP path에서는 빠진다는 사실도 스스로 알 수 없었다.</p>
</div>

<div class="bilingual">
  <p class="en">Before I created the statistical context, it also could not tell that the “interesting result” was mostly noise. So I ended up drawing a clearer line: static analysis answers local questions, dynamic analysis answers system questions. Good debugging needs both.</p>
  <p class="ko">내가 먼저 통계적 맥락을 만들어주기 전에는 “흥미로운 결과”가 사실 노이즈라는 판단도 해주지 못했다. 그래서 지금은 선을 조금 더 분명하게 긋게 됐다. 정적 분석은 로컬한 질문에 답하고, 동적 분석은 시스템 수준의 질문에 답한다. 좋은 디버깅에는 둘 다 필요하다.</p>
</div>

<div class="bilingual">
  <h2 class="en">This is not an anti-AI story</h2>
  <h2 class="ko">이건 AI가 별로라는 이야기가 아니다</h2>
</div>

<div class="bilingual">
  <p class="en">If anything, AI is what made this cycle practical. Without automation, I would have stopped much earlier. I might even have believed a clean story the data never really supported. Fifty-eight controlled runs took about five hours, 72 TTL sweep runs took about six hours, and 28 reruns after the fixes took about two and a half hours. If all of that had been manual, I probably would not have done it.</p>
  <p class="ko">오히려 반대다. 이번 사이클을 끝까지 밀어붙일 수 있었던 건 AI 덕분에 실험 자동화가 가능했기 때문이다. 자동화가 없었다면 나는 훨씬 일찍 멈췄을 것이다. 데이터가 허락하지도 않는 깔끔한 서사를 스스로 믿고 끝냈을 가능성도 높다. 58회 통제 실험은 약 5시간, 72회 TTL sweep은 약 6시간, 수정 후 28회 재실험은 약 2시간 30분이 걸렸다. 이걸 수동으로 해야 했다면, 애초에 시작하지 않았을 가능성이 크다.</p>
</div>

<div class="bilingual">
  <p class="en">That changes debugging culture. If rerunning a real experiment costs two days, people guess. If it costs a few hours, people rerun. AI automated a big part of the coding, and the load-test pipeline automated a big part of the experimentation. I needed both loops closed at the same time.</p>
  <p class="ko">이 차이는 디버깅 문화 자체를 바꾼다. 진짜 실험을 다시 돌리는 데 이틀이 걸리면 사람은 추정하게 된다. 반대로 몇 시간이면 끝난다면, 한 번 더 돌려서 확인하게 된다. AI는 코딩 작업의 큰 부분을 자동화했고, 부하 실험 파이프라인은 실험 작업의 큰 부분을 자동화했다. 이번 루프를 끝까지 닫는 데는 둘 다 필요했다.</p>
</div>

<div class="bilingual">
  <p class="en">Would this have exploded in production right away? Probably not, if I had stayed on the default settings. But that is not the interesting part. Both bugs lived in fallback or “more accurate looking” configurations. The real risk was optimization.</p>
  <p class="ko">그렇다면 이 버그가 실제 운영에서 곧바로 터졌을까? 솔직히 말하면 기본 설정 그대로였다면 바로 터지지는 않았을 가능성이 크다. 하지만 중요한 건 다른 데 있다. 두 버그 모두 fallback 환경이나 “더 정확해 보이는” 설정에서 발현되는 문제였다는 점이다. 진짜 위험은 최적화였다.</p>
</div>

<div class="bilingual">
  <p class="en">If I had read the code, decided admitted counting looked cleaner, and enabled it before a high-load event, I would have put a path into production that looked rational on paper and was broken in reality. That is the practical value of experiment-driven debugging: it filters out optimizations that are actually failure paths before users ever see them.</p>
  <p class="ko">코드를 읽고 admitted counting이 더 정확해 보인다고 판단한 뒤, 고부하 이벤트 전에 그 설정을 켰다면 종이 위에서는 합리적으로 보이는 변경을 실제로는 망가진 경로째 운영에 넣었을 가능성이 높다. 실험 기반 디버깅의 실질적인 가치는 바로 여기에 있다. 더 잘해보겠다고 손댄 최적화가 실제로는 장애 경로인지, 운영 전에 걸러낼 수 있다는 점이다.</p>
</div>

<div class="bilingual">
  <h2 class="en">What a useful debugging loop looked like</h2>
  <h2 class="ko">실제로 효과가 있었던 디버깅 루프</h2>
</div>

<div class="bilingual">
  <p class="en">If I had trusted code review alone, I probably would have ended with a plausible explanation and the wrong fix. What actually worked was a different loop.</p>
  <p class="ko">코드 리뷰만 믿고 갔다면, 아마 그럴듯한 설명과 틀린 수정안을 들고 끝났을 것이다. 실제로 효과가 있었던 건 다른 루프였다.</p>
</div>

<div class="bilingual">
  <div class="en">
    <ol>
      <li>Run experiments until the system produces a concrete question.</li>
      <li>Reread the code with that question in mind.</li>
      <li>Go back and verify the answer on the real system.</li>
    </ol>
  </div>
  <div class="ko">
    <ol>
      <li>실험을 돌려서 시스템이 먼저 구체적인 질문을 만들게 한다.</li>
      <li>그 질문을 들고 코드를 다시 읽는다.</li>
      <li>마지막으로 실제 시스템에서 그 답이 맞는지 확인한다.</li>
    </ol>
  </div>
</div>

<div class="bilingual">
  <p class="en">AI sped up the coding side. Automation sped up the experimental side. Neither replaced the other. Static analysis did not lose to dynamic analysis. It needed dynamic analysis to tell it where to look.</p>
  <p class="ko">AI는 코딩 쪽을 빠르게 만들었고, 자동화는 실험 쪽을 빠르게 만들었다. 둘 중 하나가 다른 하나를 대체한 게 아니었다. 정적 분석이 동적 분석에게 “진 것”도 아니다. 정적 분석은 어디를 봐야 하는지 알려주는 동적 분석이 필요했다.</p>
</div>

<div class="bilingual">
  <p class="en">People keep asking whether AI replaces developers. I think the more useful question is different: how should developers use AI, and in what order?</p>
  <p class="ko">사람들은 아직도 AI가 개발자를 대체하느냐 마느냐를 이야기한다. 하지만 지금 더 중요한 질문은 그게 아니다. 개발자는 AI를 어떻게, 어떤 순서와 맥락으로 써야 하는가.</p>
</div>
