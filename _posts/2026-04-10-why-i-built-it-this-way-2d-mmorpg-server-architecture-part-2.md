---
title: "Why I Built It This Way: Design Decisions Behind a 2D MMORPG Server Architecture (Part 2)"
title_kr: "왜 이렇게 만들었나: 2D MMORPG 서버 아키텍처 설계 의사결정기 (하)"
date: 2026-04-10 12:00:00 +0800
categories:
  - server-engineering
excerpt_en: "Part 2 of the architecture log: how the server survives load, where it actually failed, why queue admission exists, why Lua DSL and OpsEngine were introduced, and what I chose to give up."
excerpt_kr: "아키텍처 기록 2편. 서버가 부하를 어떻게 버티는지, 실제로 어디서 터졌는지, 왜 Queue Admission과 Lua DSL, OpsEngine을 넣었는지, 그리고 무엇을 의식적으로 포기했는지 정리한 글."
---

<div class="bilingual">
  <p class="en lede">At 15,000 concurrent users, one node died. Five thousand players rushed into the remaining three nodes in under a second, and the cluster collapsed in sequence. The root cause was a five-second heartbeat. Another time, one line of chat logging disconnected 14,510 users. There are lessons you only learn while the server is actually dying in front of you.</p>
  <p class="ko lede">동접 15,000에서 서버 하나가 죽었다. 5,000명이 1초 안에 나머지 3개 노드로 몰려들었고, 클러스터가 연쇄적으로 무너졌다. 원인은 하트비트 주기 5초. 또 한 번은 채팅 로그 한 줄 때문에 14,510명이 튕겼다. 아키텍처 문서에는 없는, 실제로 서버가 죽으면서 배운 것들이 있다.</p>
</div>

<div class="bilingual">
  <p class="en">Part 1 focused on the base structure: the hack of repurposing Nakama's ephemeral matches into persistent MMORPG maps, the split into GAME, WORLD, and CONTROL, PostgreSQL with Citus, Redis as both authority and cache, WorldRPC's four lanes, and the Single Writer pattern. Those were answers to the question, "What shape should this system take?"</p>
  <p class="ko">1편에서는 서버의 기반 구조를 다뤘다. Nakama의 휘발성 매치를 영속 MMORPG 맵으로 전용한 "꼼수", 3계층 분리, PostgreSQL + Citus 샤딩, Redis 이중 역할, WorldRPC 4차선, Single Writer 패턴. 이것들은 "어떤 구조로 만들 것인가"에 대한 답이었다.</p>
</div>

<div class="bilingual">
  <p class="en">Part 2 answers a different question: <strong>how does this structure survive under load?</strong> And the more honest version: <strong>where did it actually break, and what did I choose to give up?</strong> I am still trying to write this in a way that mid-level and senior engineers outside game development can follow. In most cases it will still be faster to paste a paragraph into an AI and ask questions. But when the problem is an architectural tradeoff that still needs a human decision, I am happy to hear from people by email.</p>
  <p class="ko">2편에서는 다른 질문에 답한다. <strong>"이 구조가 부하에서 어떻게 버티는가."</strong> 그리고 더 솔직한 질문. <strong>"어디서 터졌고, 뭘 포기했는가."</strong> 이 글은 미드-시니어 정도의 개발자라면 가능한 쉽게 읽을 수 있도록 설명하는 것을 목표로 한다. 대부분의 경우에는 궁금한 것이 있다고 하더라도 글을 긁어서 그냥 AI에게 물어보는 것이 더 빠를 것이다. 그러나 설계 디테일이나 구상하고 있는 아키텍처의 트레이드오프에 대한 의사결정이 필요한 경우에는 아직까지는 AI의 도움이 크지 않을 수 있으니, 그러한 경우에는 메일을 통한 문의 등을 환영한다.</p>
</div>

<div class="bilingual">
  <p class="en">If you have not read Part 1 yet, it is here: <a href="/server-engineering/2026/04/09/why-i-built-it-this-way-2d-mmorpg-server-architecture-part-1/">Why I Built It This Way (Part 1)</a>.</p>
  <p class="ko">1편을 아직 읽지 않았다면 여기서 먼저 볼 수 있다: <a href="/server-engineering/2026/04/09/why-i-built-it-this-way-2d-mmorpg-server-architecture-part-1/">왜 이렇게 만들었나 (상)</a>.</p>
</div>

---

<div class="bilingual">
  <h2 class="en">Queue Admission: why five stages of entry control exist</h2>
  <h2 class="ko">Queue Admission: 왜 5단계 입장 제어인가</h2>
</div>

<div class="bilingual">
  <h3 class="en">The server dies before it is technically full</h3>
  <h3 class="ko">서버는 "가득 차기 전에" 죽는다</h3>
</div>

<div class="bilingual">
  <p class="en">At first I set the heartbeat interval to five seconds. Then the server collapsed.</p>
  <p class="ko">처음엔 하트비트를 5초로 잡았다. 그리고 서버가 무너졌다.</p>
</div>

<div class="bilingual">
  <p class="en">At 15,000 CCU, one node restarted. Five thousand players attempted to reconnect to the remaining three nodes in under a second. That meant 5,000 simultaneous session creations, match assignments, and WorldRPC bootstrap calls. The other three nodes then died in sequence. This is the classic <strong>thundering herd</strong> problem.</p>
  <p class="ko">동접 15,000 상태에서 노드 하나가 재시작됐다. 5,000명이 1초 안에 나머지 3개 노드로 재접속을 시도했다. 5,000명이 동시에 세션 생성, 매치 배정, WorldRPC 부트스트랩을 요청하자 나머지 3개 노드도 연쇄적으로 죽었다. 이게 <strong>Thundering Herd</strong>(우르르 몰려가기)다.</p>
</div>

<div class="bilingual">
  <p class="en">In a four-node cluster, if each match holds one hundred players and each node owns fifty matches, the theoretical capacity is 20,000 users. But the 20,001st user was not the real problem. The cluster was dying long before it reached its steady-state limit, because reconnect storms hit it faster than it could absorb them.</p>
  <p class="ko">4노드 클러스터에서 매치당 100명, 노드당 50개 매치라면 이론적 최대치는 20,000명이다. 하지만 20,001번째 플레이어가 문제가 아니었다. 서버는 "가득 찬" 게 아니라, 가득 차기도 전에 재접속 폭풍에 죽었다.</p>
</div>

> **Same problem on the web:** cache stampedes when a hot key expires, or limited sales where tens of thousands of users arrive at once. The usual answers are request limiting, centralized rate limiting, or a virtual waiting room.
>
> **Same answer in this server:** if you do not control admission speed, the server dies before it is full.

<div class="bilingual">
  <h3 class="en">Decision</h3>
  <h3 class="ko">결정</h3>
</div>

<div class="bilingual">
  <p class="en">I split player admission into five gates.</p>
  <p class="ko">클라이언트 접속을 5단계 게이트로 나눴다.</p>
</div>

```text
┌─────────────────────────────────────────┐
│ Gate 1: Connection Admission           │
│ TCP-level concurrent connection limit  │
│ On overflow: reject immediately        │
│ Web analogy: Nginx worker_connections  │
├─────────────────────────────────────────┤
│ Gate 2: Queue Gate                     │
│ Shared Redis gate, operator can block  │
│ On overflow: enter waiting queue       │
│ Web analogy: rate limiter / waiting room│
├─────────────────────────────────────────┤
│ Gate 3: Match Entry Queue              │
│ Redis FIFO, node-local batch consumer  │
│ Admit 10 users every 100ms             │
│ Web analogy: queue consumer batch size │
├─────────────────────────────────────────┤
│ Gate 4: Match Dispatch                 │
│ Warm pool or on-demand match assignment│
│ Web analogy: discovery + connection pool│
├─────────────────────────────────────────┤
│ Gate 5: Session Setup                  │
│ WorldRPC bootstrap for player session  │
│ Web analogy: post-login profile load   │
└─────────────────────────────────────────┘
```

<div class="bilingual">
  <h3 class="en">Why each gate exists</h3>
  <h3 class="ko">각 게이트의 설계 이유</h3>
</div>

<div class="bilingual">
  <p class="en"><strong>Gate 1: Connection Admission.</strong> This is the outermost defense line. It caps concurrent TCP connections before the client consumes any meaningful game-server resources. If a connection fails here, the server does not even spend time on a WebSocket handshake.</p>
  <p class="ko"><strong>Gate 1: Connection Admission.</strong> 가장 바깥쪽 방어선이다. TCP 레벨에서 동시접속 수를 제한한다. 이 게이트를 통과하지 못한 클라이언트는 서버 자원을 전혀 소모하지 않는다. WebSocket 핸드셰이크조차 일어나지 않는다.</p>
</div>

<div class="bilingual">
  <p class="en"><strong>Gate 2: Queue Gate.</strong> This is the real control point. Every node sees the same Redis-backed gate, and operators can manually block admission during incidents. I intentionally did not make it per-node. A per-node gate creates the same problem as per-instance rate limits on the web: total capacity silently scales with the number of servers. Admission speed has to be controlled at the cluster level.</p>
  <p class="ko"><strong>Gate 2: Queue Gate.</strong> 핵심 게이트다. Redis에 공유 상태를 두고 모든 노드가 같은 게이트를 본다. 운영자는 장애 중에 수동으로 입장을 막을 수도 있다. 왜 노드별이 아니라 공유 상태냐면, 웹 서비스에서 Rate Limiter를 각 인스턴스에 두면 인스턴스 수만큼 허용량이 늘어나는 것과 같은 문제가 생기기 때문이다. 클러스터 전체의 입장 속도를 하나의 게이트로 제어해야 한다.</p>
</div>

<div class="bilingual">
  <p class="en"><strong>Gate 3: Match Entry Queue.</strong> This is the batching layer. I let ten users through every 100ms, so roughly one hundred per second. Why not drain the queue all at once? Because cold match creation still costs 100-400ms depending on map initialization. Bursting too many new admissions at once would freeze a node in exactly the wrong moment.</p>
  <p class="ko"><strong>Gate 3: Match Entry Queue.</strong> 배치 처리의 핵심이다. 100ms마다 10명씩 입장시킨다. 1초에 100명 정도다. 왜 한 번에 다 안 넣나? Warm Pool에 없는 맵을 새로 만들 때는 여전히 100-400ms가 걸린다. 그 순간 입장을 몰아주면 노드가 바로 멈춘다.</p>
</div>

<div class="bilingual">
  <p class="en"><strong>Gate 4: Match Dispatch.</strong> This gate mixes two strategies. If a map already exists in the warm pool, I reuse it. If not, I create a fresh match on demand. It is the same shape as using a warm connection pool first and opening new connections only when necessary.</p>
  <p class="ko"><strong>Gate 4: Match Dispatch.</strong> 두 가지 전략을 같이 쓴다. 미리 만들어 둔 Warm Pool에서 꺼낼 수 있으면 거기서 쓰고, 없으면 그때 새 매치를 만든다. 웹에서 먼저 커넥션 풀의 유휴 커넥션을 쓰고, 부족할 때만 새 커넥션을 여는 것과 같은 구조다.</p>
</div>

<div class="bilingual">
  <p class="en"><strong>Gate 5: Session Setup.</strong> Entering a match does not mean a player is fully ready. The server still needs inventory, equipment, cosmetics, and status from WORLD. So I use <strong>Lazy Join</strong>: load only the actor ID and spawn coordinates synchronously, then mark the player as <code>NeedsSessionSetup</code> and finish the rest over the next few match-loop ticks.</p>
  <p class="ko"><strong>Gate 5: Session Setup.</strong> 매치에 들어왔다고 바로 플레이 가능한 게 아니다. 인벤토리, 장비, 외형, 상태값을 WORLD에서 받아와야 한다. 그래서 <strong>Lazy Join</strong>을 쓴다. actorID와 스폰 좌표만 동기로 로드하고, 나머지는 <code>NeedsSessionSetup</code> 플래그를 켜서 다음 MatchLoop 틱들에서 비동기로 가져온다.</p>
</div>

<div class="bilingual">
  <p class="en">The player appears on the map immediately. The heavy parts arrive a few ticks later. It is the same user-experience choice as rendering the page shell first and lazy-loading the rest.</p>
  <p class="ko">플레이어는 즉시 맵에 나타나고, 무거운 부분은 몇 틱 뒤에 준비된다. 페이지 껍데기를 먼저 보여주고 나머지를 lazy loading하는 것과 같은 UX 선택이다.</p>
</div>

<div class="bilingual">
  <h3 class="en">Measured difference</h3>
  <h3 class="ko">성능 차이</h3>
</div>

<div class="bilingual">
  <p class="en">On the real cluster, with thousands of synthetic clients attached, the difference looked like this.</p>
  <p class="ko">실제 클러스터에 테스트 클라이언트를 수천 개 붙여서 측정한 결과는 이랬다.</p>
</div>

```text
Queue Gate ON:  p99 latency 5.9ms
Queue Gate OFF: p99 latency 990ms
                (about 170x worse)
```

<div class="bilingual">
  <p class="en">With the gate turned off, reconnect storms saturate shared resources like DB connections and Redis pipelines. With the gate turned on, throughput is intentionally flattened. This is a conscious tradeoff: I delay admission so I can protect everyone who is already in the game.</p>
  <p class="ko">Queue Gate가 없으면 재접속 폭풍이 DB 커넥션과 Redis 파이프라인 같은 공유 자원을 한 번에 포화시킨다. Gate가 있으면 처리량을 일부러 평평하게 만든다. 이건 의도적인 선택이다. 입장을 늦추는 대신, 이미 플레이 중인 유저들의 경험을 지킨다.</p>
</div>

<div class="bilingual">
  <h3 class="en">Idempotency and duplicate suppression</h3>
  <h3 class="ko">멱등성과 중복 방지</h3>
</div>

<div class="bilingual">
  <p class="en">The network is unreliable. A client can get a token at Gate 2, disconnect on the way to Gate 3, and then retry. If I do not suppress duplicates, the queue explodes and one person can end up occupying two positions.</p>
  <p class="ko">네트워크는 불안정하다. 클라이언트가 Gate 2에서 토큰을 받고 Gate 3으로 넘어가다가 연결이 끊길 수 있다. 중복 요청을 막지 않으면 큐가 폭발하고, 한 명의 유저가 대기열에서 두 자리를 차지할 수 있다.</p>
</div>

```lua
-- Why this matters: duplicate check and enqueue must be atomic.
-- Otherwise the same request can slip in twice between them.
local existing = redis.call('GET', KEYS[1])
if existing then
    local rank = redis.call('ZRANK', KEYS[2], existing)
    return {existing, rank}
end

local seq = redis.call('INCR', KEYS[3])
redis.call('ZADD', KEYS[2], seq, ARGV[1])
redis.call('SETEX', KEYS[1], ARGV[3], ARGV[1])
return {ARGV[1], seq}
```

<div class="bilingual">
  <p class="en">This is the same idea as the <strong>idempotency key</strong> pattern in payment APIs. One request should own one logical slot, even if the transport retries it several times.</p>
  <p class="ko">이건 결제 API의 <strong>멱등성 키</strong> 패턴과 같은 발상이다. 전송이 여러 번 재시도되더라도, 하나의 요청은 하나의 논리적 슬롯만 가져야 한다.</p>
</div>

<div class="bilingual">
  <h3 class="en">The queue itself becomes an attack surface</h3>
  <h3 class="ko">대기열 자체가 공격 대상이 된다</h3>
</div>

<div class="bilingual">
  <p class="en">Once you centralize admission, that central point becomes attractive to attackers. A poisoned queue can deny service to normal players before they even touch the game. I am omitting the implementation details here, but the operating principle is simple: make the attacker pay the cost of admission validation, not the server.</p>
  <p class="ko">입장 제어를 한 곳으로 모으면, 그 한 곳이 공격 대상이 된다. 큐를 허위 요청으로 오염시키면 정상 유저는 게임에 닿기도 전에 막힌다. 구체적인 구현은 생략하지만, 운영 원칙은 단순하다. 입장 검증 비용을 서버가 아니라 공격자 쪽에 떠넘겨야 한다.</p>
</div>

<div class="bilingual">
  <h3 class="en">Five-second heartbeat: the cluster collapsed</h3>
  <h3 class="ko">하트비트 5초. 서버가 무너졌다.</h3>
</div>

<div class="bilingual">
  <p class="en">The first Queue Admission implementation still used a five-second heartbeat. In load tests it created a self-reinforcing loop.</p>
  <p class="ko">Queue Admission을 처음 구현했을 때도 하트비트 주기는 5초였다. 부하 테스트에서 자기 강화 루프가 터졌다.</p>
</div>

```text
5s heartbeat interval
  -> Redis session count lags behind reality
  -> Gate thinks there is spare room
  -> Gate admits too many users
  -> server overloads and reconnect storm grows
  -> Gate again thinks there is spare room
  -> loop repeats
```

<div class="bilingual">
  <p class="en">One node over-admitted by 475 users while another under-admitted by 216. Reconnect attempts climbed toward 15,000. After reducing the heartbeat to one second, the worst error fell from 330 users to 66 and the loop stopped feeding itself. What felt "fast enough" at design time turned out to be slow enough to cause the failure.</p>
  <p class="ko">한 노드에 +475명이 과다 입장하고, 다른 노드에서는 -216명이 부족한 불균형이 생겼다. 재접속은 거의 15,000건까지 치솟았다. 하트비트를 1초로 줄이자 최대 오차가 330명에서 66명으로 떨어졌고, 루프가 끊어졌다. 설계할 때는 "충분히 빠르다"고 생각했던 주기가 실제로는 장애 원인이었다.</p>
</div>

<div class="bilingual">
  <h3 class="en">One chat log line disconnected 14,510 users</h3>
  <h3 class="ko">로그 한 줄. 14,510명이 튕겼다.</h3>
</div>

<div class="bilingual">
  <p class="en">During a 3K CCU cold-start test, I saw 14,510 reconnects. The root cause turned out to be one diagnostic chat log line. It was serializing JSON and writing debug output around 250 times per second. That overhead delayed the match loop enough to trigger pong timeouts.</p>
  <p class="ko">3K CCU 콜드 스타트 테스트에서 재접속 14,510건이 나왔다. 원인을 추적했더니 채팅 진단 로그 한 줄이었다. 초당 250건 수준의 채팅 메시지마다 JSON 직렬화를 포함한 디버그 로그를 찍고 있었고, 그 오버헤드가 매치 루프를 밀어서 pong 타임아웃을 만들었다.</p>
</div>

<div class="bilingual">
  <p class="en">Removing that log took reconnects from 14,510 down to 3,537, a 75.6 percent improvement. Not every performance problem is architectural. Sometimes it really is one line of logging in the wrong place.</p>
  <p class="ko">그 로그를 제거하자 14,510건에서 3,537건으로 줄었다. 75.6% 개선이다. 성능 문제의 원인이 항상 아키텍처에 있는 건 아니다. 가끔은 정말 로그 한 줄이 문제다.</p>
</div>

> **Core lesson of this section:** the cluster dies before it is full if admission is unmanaged, and tiny details like heartbeat cadence or one debug log line can be enough to push it over.

---

<div class="bilingual">
  <h2 class="en">Lua DSL: why content had to be separated from code</h2>
  <h2 class="ko">Lua DSL: 게임 콘텐츠를 코드에서 분리한 이유</h2>
</div>

<div class="bilingual">
  <h3 class="en">NPC ticks sent RPCs directly, and WORLD died</h3>
  <h3 class="ko">NPC 틱에서 RPC를 직접 보냈다. WORLD가 죽었다.</h3>
</div>

<div class="bilingual">
  <p class="en">At first, all NPC logic lived inside Go. If an NPC dropped an item, it called WorldRPC directly. One hundred NPCs running at 20Hz generate 2,000 RPCs per second. WORLD died under that pressure.</p>
  <p class="ko">처음에는 NPC 로직이 전부 Go 코드 안에 있었다. NPC가 아이템을 드롭하면 바로 WorldRPC를 호출했다. NPC 100마리가 20Hz로 틱을 돌리면 초당 2,000건의 RPC가 나온다. WORLD가 죽었다.</p>
</div>

<div class="bilingual">
  <p class="en">There was a more basic problem too. If I wanted to change one line of NPC dialogue, I had to edit Go, rebuild the plugin, and restart the server. If I wanted to change a quest reward from 100 gold to 150, same process. That is content living in the wrong layer.</p>
  <p class="ko">더 근본적인 문제도 있었다. NPC가 "안녕하세요"라는 대사를 바꾸려면 Go 코드를 수정하고, 플러그인을 다시 빌드하고, 서버를 재시작해야 했다. 퀘스트 보상을 100골드에서 150골드로 바꿀 때도 같은 과정이었다. 콘텐츠가 잘못된 레이어에 올라가 있던 셈이다.</p>
</div>

> **Same problem on the web:** if changing banner copy requires a full deployment, the CMS layer is missing.
>
> **Same answer in this server:** quest logic, dialogue, and small behavior scripts belong in a content layer, not in the engine binary.

<div class="bilingual">
  <h3 class="en">Decision</h3>
  <h3 class="ko">결정</h3>
</div>

<div class="bilingual">
  <p class="en">I moved NPC event handlers into Lua scripts and embedded a Lua VM inside the server.</p>
  <p class="ko">NPC 이벤트 핸들러를 Lua 스크립트로 분리하고, 서버 안에 Lua VM을 넣었다.</p>
</div>

```lua
function on_click(ctx)
    if ctx.player:has_item("quest_letter") then
        ctx.npc:say("You brought the letter. Here is your reward.")
        ctx.player:grant_item("gold", 150)
        ctx.player:complete_quest("delivery_quest")
    else
        ctx.npc:say("Please bring me the letter from the king.")
    end
end
```

<div class="bilingual">
  <p class="en">The reason for Lua specifically was practical: <strong>gopher-lua</strong> is a pure-Go VM. No CGo, no awkward runtime boundary, and it embeds naturally into the existing process.</p>
  <p class="ko">굳이 Lua를 쓴 이유도 실용적이다. <strong>gopher-lua</strong>는 순수 Go Lua VM이다. CGo 없이 Go 바이너리 안에 들어가고, 기존 프로세스와 자연스럽게 붙는다.</p>
</div>

<div class="bilingual">
  <h3 class="en">Three permission tiers</h3>
  <h3 class="ko">3계층 권한 모델</h3>
</div>

<div class="table-wrap bilingual">
  <table class="en">
    <thead>
      <tr>
        <th>Permission</th>
        <th>Meaning</th>
        <th>Examples</th>
      </tr>
    </thead>
    <tbody>
      <tr>
        <td><strong>Read-Only</strong></td>
        <td>Can only inspect state</td>
        <td><code>player:get_hp()</code>, <code>npc:get_pos()</code></td>
      </tr>
      <tr>
        <td><strong>Live Mutate</strong></td>
        <td>Can change in-memory game state</td>
        <td><code>player:set_hp(100)</code>, <code>npc:move_to(x,y)</code></td>
      </tr>
      <tr>
        <td><strong>Persistent</strong></td>
        <td>Can trigger durable state change</td>
        <td><code>player:grant_item()</code>, <code>player:add_gold()</code></td>
      </tr>
    </tbody>
  </table>

  <table class="ko">
    <thead>
      <tr>
        <th>권한</th>
        <th>설명</th>
        <th>예시</th>
      </tr>
    </thead>
    <tbody>
      <tr>
        <td><strong>Read-Only</strong></td>
        <td>상태를 읽기만 함</td>
        <td><code>player:get_hp()</code>, <code>npc:get_pos()</code></td>
      </tr>
      <tr>
        <td><strong>Live Mutate</strong></td>
        <td>메모리 상태를 변경</td>
        <td><code>player:set_hp(100)</code>, <code>npc:move_to(x,y)</code></td>
      </tr>
      <tr>
        <td><strong>Persistent</strong></td>
        <td>DB에 영속적 변경</td>
        <td><code>player:grant_item()</code>, <code>player:add_gold()</code></td>
      </tr>
    </tbody>
  </table>
</div>

<div class="bilingual">
  <p class="en">Those permissions are not universal. They depend on the execution context.</p>
  <p class="ko">그리고 이 권한은 항상 같은 것이 아니다. 실행 컨텍스트마다 허용 범위가 다르다.</p>
</div>

<div class="table-wrap bilingual">
  <table class="en">
    <thead>
      <tr>
        <th>Context</th>
        <th>Read</th>
        <th>Live Mutate</th>
        <th>Persistent</th>
        <th>Timeout</th>
      </tr>
    </thead>
    <tbody>
      <tr>
        <td><code>npc_click</code></td>
        <td>Yes</td>
        <td>Yes</td>
        <td>Yes</td>
        <td>500ms</td>
      </tr>
      <tr>
        <td><code>npc_tick</code></td>
        <td>Yes</td>
        <td>Mostly yes</td>
        <td><strong>Forbidden</strong></td>
        <td>100ms</td>
      </tr>
      <tr>
        <td><code>combat</code></td>
        <td>Yes</td>
        <td>Yes</td>
        <td>Only a small allowlist</td>
        <td>200ms</td>
      </tr>
    </tbody>
  </table>

  <table class="ko">
    <thead>
      <tr>
        <th>컨텍스트</th>
        <th>Read</th>
        <th>Live Mutate</th>
        <th>Persistent</th>
        <th>타임아웃</th>
      </tr>
    </thead>
    <tbody>
      <tr>
        <td><code>npc_click</code></td>
        <td>O</td>
        <td>O</td>
        <td>O</td>
        <td>500ms</td>
      </tr>
      <tr>
        <td><code>npc_tick</code></td>
        <td>O</td>
        <td>대부분 O</td>
        <td><strong>금지</strong></td>
        <td>100ms</td>
      </tr>
      <tr>
        <td><code>combat</code></td>
        <td>O</td>
        <td>O</td>
        <td>일부 allowlist만</td>
        <td>200ms</td>
      </tr>
    </tbody>
  </table>
</div>

<div class="bilingual">
  <h3 class="en">Why <code>npc_tick</code> forbids persistence</h3>
  <h3 class="ko">왜 <code>npc_tick</code>에서 Persistent를 금지하는가</h3>
</div>

<div class="bilingual">
  <p class="en">This is the single most important rule in the DSL layer, and it came directly from failure. NPC ticks run at 20Hz. With one hundred NPCs, that means 2,000 executions per second. If each of them can emit persistent writes or synchronous WorldRPC calls, the content layer becomes a denial-of-service tool against WORLD.</p>
  <p class="ko">이게 DSL 계층에서 가장 중요한 규칙이고, 실패에서 바로 나온 규칙이다. NPC 틱은 20Hz로 돈다. NPC 100마리면 초당 2,000번 실행된다. 여기서 영속 쓰기나 동기 WorldRPC를 허용하면, 콘텐츠 레이어가 그대로 WORLD를 공격하는 도구가 된다.</p>
</div>

```text
100 NPCs x 20 ticks/sec = 2,000 executions/sec
If each tick can write persistently:
  -> 2,000 durable operations/sec from NPC logic alone
  -> WORLD becomes the bottleneck immediately
```

<div class="bilingual">
  <p class="en">So <code>npc_tick</code> is allowed to observe state, compute, and mutate local state, but not to perform durable writes. When an NPC wants to do something durable, it writes an <strong>intent</strong> into a buffer instead.</p>
  <p class="ko">그래서 <code>npc_tick</code>은 상태를 읽고, 계산하고, 로컬 상태를 바꾸는 것까지만 허용한다. 영속 쓰기는 금지한다. NPC가 영속적인 일을 하고 싶으면 대신 <strong>intent</strong>를 버퍼에 적는다.</p>
</div>

```lua
function on_tick(ctx)
    if ctx.npc:get_hp() <= 0 then
        ctx.npc:queue_intent("drop_loot", {item="sword", qty=1})
    end
end

-- Later, the match loop dispatches queued intents
-- and converts them into deferred async RPCs.
```

<div class="bilingual">
  <p class="en">This is the same pattern as queueing email work instead of sending email in the request path. The tick records what should happen. The engine chooses when to execute it safely.</p>
  <p class="ko">이건 요청 처리 중에 이메일을 직접 보내지 않고 큐에 적재하는 것과 같은 패턴이다. 틱은 "무엇을 해야 하는가"만 기록하고, 실제 실행 시점은 엔진이 안전하게 선택한다.</p>
</div>

<div class="bilingual">
  <h3 class="en">Sandboxing</h3>
  <h3 class="ko">샌드박스</h3>
</div>

<div class="bilingual">
  <p class="en">A content script should never be able to kill the server. So the Lua environment strips out dangerous functions and modules entirely.</p>
  <p class="ko">콘텐츠 스크립트가 서버를 죽일 수 있으면 안 된다. 그래서 Lua 환경에서 위험한 함수와 모듈은 아예 제거했다.</p>
</div>

```go
disabled := []string{
    "dofile", "loadfile", "load", "loadstring",
    "pcall", "xpcall",
    "rawset", "rawget", "rawequal",
    "setmetatable", "getmetatable",
    "collectgarbage", "newproxy",
    "string.dump",
}
disabledModules := []string{"os", "io", "debug", "package"}
```

<div class="bilingual">
  <p class="en">Each context also has a hard execution timeout. If a script loops forever inside <code>npc_tick</code>, the VM is interrupted before it can freeze the server. This is another explicit tradeoff: I give up some debugging convenience in return for content independence and operational safety.</p>
  <p class="ko">각 컨텍스트에는 실행 시간 제한도 있다. <code>npc_tick</code> 안에서 무한루프가 돌아도 VM이 먼저 끊어지기 때문에 서버 전체가 멈추지 않는다. 이것도 명시적인 트레이드오프다. 디버깅 편의성 일부를 포기하는 대신 콘텐츠 독립성과 운영 안정성을 얻는다.</p>
</div>

> **Core lesson of this section:** content should behave more like data than engine code, and data needs fences so it cannot take the engine down with it.

---

<div class="bilingual">
  <h2 class="en">OpsEngine: why operational judgment became an engine instead of scattered if-statements</h2>
  <h2 class="ko">OpsEngine: 왜 운영 판단을 코드 분기가 아니라 엔진으로 만들었는가</h2>
</div>

<div class="bilingual">
  <h3 class="en">Ten <code>if</code> statements in ten files missed a composite failure</h3>
  <h3 class="ko">if문 10개가 흩어져 있었다. 그리고 복합 장애를 놓쳤다.</h3>
</div>

<div class="bilingual">
  <p class="en">At the beginning, operational judgment was just code branches scattered around the system.</p>
  <p class="ko">처음에는 운영 판단을 코드 분기로 넣었다.</p>
</div>

```go
if dbLatencyP99 > 500*time.Millisecond {
    closeAdmission()
}
if redisErrorRate > 0.05 {
    disablePartyFeature()
}
```

<div class="bilingual">
  <p class="en">The problem is not that these checks are wrong. The problem is that they do not know about each other. A day where the DB is somewhat slow and Redis is somewhat unhealthy may still look "below threshold" to each isolated rule, even though the combination is clearly dangerous.</p>
  <p class="ko">문제는 이런 체크가 틀렸다는 게 아니다. 서로를 모른다는 게 문제다. DB가 조금 느리고 Redis도 조금 불안한 날이 있을 수 있는데, 각각은 임계치 아래라서 괜찮아 보이더라도 조합하면 분명히 위험한 상황이 생긴다.</p>
</div>

<div class="bilingual">
  <p class="en">Once those decisions are scattered, the system can no longer answer a basic question in one place: <strong>what state is the server in right now?</strong></p>
  <p class="ko">판단 로직이 흩어지는 순간, 시스템은 가장 기본적인 질문에 한 곳에서 답하지 못한다. <strong>지금 서버가 어떤 상태인가?</strong></p>
</div>

> **Same problem on the web:** health judgment living in controllers, middlewares, and one-off background checks instead of a centralized readiness model.

<div class="bilingual">
  <h3 class="en">Decision</h3>
  <h3 class="ko">결정</h3>
</div>

<div class="bilingual">
  <p class="en">I introduced an engine whose job is only to gather signals, evaluate policy, and propose or apply restrictions.</p>
  <p class="ko">신호를 모으고, 정책을 평가하고, 제한 조치를 제안하거나 적용하는 역할만 맡는 엔진을 만들었다.</p>
</div>

```text
                    ┌─────────────┐
                    │  OpsEngine  │
                    └──────┬──────┘
                           │
        ┌──────────────────┼──────────────────┐
        ▼                  ▼                  ▼
  ┌───────────┐    ┌───────────┐    ┌───────────┐
  │ Admission │    │Simulation │    │Persistence│
  │ Provider  │    │ Provider  │    │ Provider  │
  └───────────┘    └───────────┘    └───────────┘
```

```go
type OpsEngine struct {
    providers []SignalProvider
    policy    *PolicyTable
    history   *TransitionLog
    applier   RestrictionApplier
}
```

<div class="bilingual">
  <p class="en">Health is evaluated by a simple rule: <strong>worst state wins.</strong> If persistence is red while simulation is green, the cluster is red. The point is not elegance. The point is making the system pessimistic in the right places.</p>
  <p class="ko">건강 상태는 단순한 규칙으로 계산한다. <strong>최악치가 전체 상태가 된다.</strong> Persistence가 red이고 Simulation이 green이어도 전체는 red다. 중요한 건 우아함이 아니라, 시스템이 필요한 곳에서 비관적으로 판단하게 만드는 것이다.</p>
</div>

```text
green  -> normal
yellow -> warn
orange -> throttle admission / restrict new matches
red    -> suggest full block / drain
```

<div class="bilingual">
  <h3 class="en">Automatic versus suggested actions</h3>
  <h3 class="ko">자동 vs 제안</h3>
</div>

<div class="table-wrap bilingual">
  <table class="en">
    <thead>
      <tr>
        <th>Action</th>
        <th>Auto-apply</th>
        <th>Reason</th>
      </tr>
    </thead>
    <tbody>
      <tr>
        <td>Queue throttle</td>
        <td>Yes</td>
        <td>Reversible and low-risk</td>
      </tr>
      <tr>
        <td>Restrict new matches</td>
        <td>Yes</td>
        <td>Protects the cluster without harming existing matches</td>
      </tr>
      <tr>
        <td>Full admission block</td>
        <td>No, suggest only</td>
        <td>Severe player-facing impact</td>
      </tr>
      <tr>
        <td>Drain node</td>
        <td>No, suggest only</td>
        <td>Needs operator judgment</td>
      </tr>
      <tr>
        <td>Safe mode</td>
        <td>Manual only</td>
        <td>Too destructive for automation</td>
      </tr>
    </tbody>
  </table>

  <table class="ko">
    <thead>
      <tr>
        <th>조치</th>
        <th>자동 적용</th>
        <th>이유</th>
      </tr>
    </thead>
    <tbody>
      <tr>
        <td>큐 게이트 스로틀</td>
        <td>O</td>
        <td>가역적이고 위험이 낮음</td>
      </tr>
      <tr>
        <td>신규 매치 제한</td>
        <td>O</td>
        <td>기존 매치를 건드리지 않고 보호 가능</td>
      </tr>
      <tr>
        <td>입장 완전 차단</td>
        <td>제안만</td>
        <td>플레이어 체감 영향이 큼</td>
      </tr>
      <tr>
        <td>드레인</td>
        <td>제안만</td>
        <td>운영자 판단이 필요함</td>
      </tr>
      <tr>
        <td>세이프 모드</td>
        <td>수동만</td>
        <td>자동화하기엔 너무 파괴적임</td>
      </tr>
    </tbody>
  </table>
</div>

<div class="bilingual">
  <p class="en">The principle is simple: <strong>orange and below may auto-apply, red and above should be proposed first.</strong> Machines are allowed to reduce pressure. Humans still decide when to take actions that feel like service interruption.</p>
  <p class="ko">원칙은 단순하다. <strong>orange 이하는 자동, red 이상은 먼저 제안.</strong> 기계는 압력을 줄이는 데까지만 자동으로 개입하고, 서비스 중단처럼 느껴지는 조치는 사람이 결정한다.</p>
</div>

<div class="bilingual">
  <h3 class="en">Audit trail</h3>
  <h3 class="ko">감사 추적</h3>
</div>

<div class="bilingual">
  <p class="en">If I cannot answer "why was admission blocked at 3 a.m. yesterday?" then the automation is not trustworthy. Every state transition is recorded.</p>
  <p class="ko">"어제 새벽 3시에 왜 입장이 막혔지?"라는 질문에 답할 수 없으면 자동화는 신뢰할 수 없다. 그래서 모든 상태 전환을 기록한다.</p>
</div>

```sql
INSERT INTO ops_audit_log (
    timestamp, previous_state, new_state,
    triggered_by, action_taken, auto_applied
)
VALUES (...);
```

<div class="bilingual">
  <p class="en">This is another explicit tradeoff. I could automate more, but I intentionally leave high-risk actions under human control. One false positive should not be able to take the service down by itself.</p>
  <p class="ko">이것도 명시적인 선택이다. 기술적으로는 더 많은 조치를 자동화할 수 있다. 하지만 위험한 조치는 의도적으로 사람 손에 남겨둔다. 오탐 한 번에 서비스가 내려가면 안 되기 때문이다.</p>
</div>

> **Core lesson of this section:** operational judgment should be centralized enough that the system can answer, in one place, what state it believes the cluster is in and why.

---

<div class="bilingual">
  <h2 class="en">Cluster design: why the default is All-in-One</h2>
  <h2 class="ko">클러스터: 왜 All-in-One인가</h2>
</div>

<div class="bilingual">
  <h3 class="en">At 22K CCU, one node took 67 percent and crashed with OOM</h3>
  <h3 class="ko">22K CCU에서 한 노드에 67%가 몰렸다. OOM으로 크래시.</h3>
</div>

<div class="bilingual">
  <p class="en">In one imbalance test, a single node ended up holding 67 percent of the cluster's users. Its Redis session TTL expired, the outer queue misread the node as empty, and the system admitted even more users there. Around 12,000 users piled into one machine, it hit OOM, and then a DNS inconsistency during container restart helped destabilize the entire cluster.</p>
  <p class="ko">불균형 테스트에서 한 노드에 전체의 67%가 몰렸다. 그 노드의 Redis 세션 TTL이 만료되자 외부 큐가 "세션 0명"으로 읽고 전원에게 입장을 허가했다. 12,000명이 한 노드에 몰려 OOM으로 크래시했고, Docker 재시작 과정의 DNS 설정 불일치까지 겹치면서 전체 클러스터가 흔들렸다.</p>
</div>

<div class="bilingual">
  <p class="en">That incident sharpened one architectural question: since the code already distinguishes GAME, WORLD, and CONTROL logically, should they be physically split as well?</p>
  <p class="ko">이 장애는 한 가지 질문을 더 선명하게 만들었다. GAME, WORLD, CONTROL을 논리적으로 나눴다면, 물리적으로도 나눠야 하는가?</p>
</div>

<div class="bilingual">
  <h3 class="en">Decision</h3>
  <h3 class="ko">결정</h3>
</div>

<div class="bilingual">
  <p class="en"><strong>The default is All-in-One.</strong> Every node runs GAME, WORLD, and CONTROL together.</p>
  <p class="ko"><strong>기본값은 All-in-One이다.</strong> 모든 노드가 GAME, WORLD, CONTROL을 다 실행한다.</p>
</div>

```text
ROLE_PROFILE=all

Node 1: [GAME + WORLD + CONTROL]
Node 2: [GAME + WORLD + CONTROL]
Node 3: [GAME + WORLD + CONTROL]
Node 4: [GAME + WORLD + CONTROL]
```

<div class="bilingual">
  <h3 class="en">Why I did not split them</h3>
  <h3 class="ko">왜 분리하지 않았는가</h3>
</div>

<div class="bilingual">
  <p class="en"><strong>First, LocalRPC is too valuable.</strong> When GAME and WORLD live in the same process, the call path is basically a function call. When they live on different nodes, the same operation becomes a network round trip that can cost 5-100ms. In a 50ms match-loop budget, ten such calls are catastrophic.</p>
  <p class="ko"><strong>첫째, LocalRPC가 너무 중요하다.</strong> GAME과 WORLD가 같은 프로세스에 있으면 사실상 함수 호출이다. 다른 노드에 있으면 같은 작업이 5-100ms의 네트워크 왕복이 된다. 50ms 틱 예산 안에서는 이런 호출이 몇 번만 쌓여도 치명적이다.</p>
</div>

<div class="bilingual">
  <p class="en"><strong>Second, operating three separately deployed services is much harder than naming three roles in one codebase.</strong> Version compatibility, rollout order, health probes, monitoring boundaries, and secret distribution all become harder. For a solo-operated service, that overhead matters.</p>
  <p class="ko"><strong>둘째, 코드베이스 안에서 역할 이름을 셋으로 나누는 것과, 배포를 셋으로 나누는 건 전혀 다른 문제다.</strong> 버전 호환성, 롤링 업데이트 순서, 헬스체크, 모니터링 경계, 시크릿 배포가 전부 더 어려워진다. 혼자 운영하는 서비스에서는 이 오버헤드가 크게 느껴진다.</p>
</div>

<div class="bilingual">
  <p class="en"><strong>Third, I reject premature physical separation.</strong> Chaos testing already validated roughly 20K-25K CCU across multiple nodes and maps, and I would only expect 60-70 percent of that in a live setting anyway. At this stage, simplicity is more valuable than theoretical future elasticity.</p>
  <p class="ko"><strong>셋째, 물리적 분리를 조기 최적화로 보았다.</strong> 카오스 테스트에서 이미 여러 노드, 여러 맵 기준 20K~25K CCU 정도는 검증했고, 실제 라이브에서는 그 60-70% 정도를 기대한다. 지금 단계에서는 미래의 이론적 확장성보다 단순성이 더 가치 있다.</p>
</div>

<div class="bilingual">
  <h3 class="en">How Redis and PostgreSQL glue the nodes together</h3>
  <h3 class="ko">Redis가 노드를 엮어주는 방법</h3>
</div>

<div class="bilingual">
  <p class="en">All-in-One does not mean nodes are isolated. PostgreSQL acts as the durable registry and Redis acts as the fast shared layer.</p>
  <p class="ko">All-in-One이라고 해서 노드가 서로 고립된다는 뜻은 아니다. PostgreSQL은 영속 레지스트리 역할을 하고, Redis는 빠른 공유 계층 역할을 한다.</p>
</div>

```text
PostgreSQL
  -> durable map instance registry
  -> "map X belongs to node 3"

Redis
  -> routing cache
  -> party state
  -> queue state
  -> cross-node notifications
```

<div class="bilingual">
  <p class="en">The pattern is very close to multi-node web infrastructure. PostgreSQL is the durable source of truth. Redis is the fast coordination plane that lets any node answer questions like "where is this map?", "who is in this party?", and "which player may enter next?"</p>
  <p class="ko">패턴 자체는 다중 웹 서버 인프라와 아주 비슷하다. PostgreSQL은 영속 정본이고, Redis는 "이 맵이 어디 있지?", "이 파티 멤버가 누구지?", "다음 입장 허가는 누구지?" 같은 질문에 어떤 노드에서든 빠르게 답하게 해주는 조정 평면이다.</p>
</div>

<div class="bilingual">
  <h3 class="en">LocalRPC versus RemoteRPC</h3>
  <h3 class="ko">LocalRPC vs RemoteRPC</h3>
</div>

```text
Same node:
  GAME -> direct handler call -> WORLD
  latency: ~0ms

Different node:
  GAME (node 1) -> HTTP POST -> WORLD (node 3)
  latency: 5-100ms
```

```go
func ControlMatchSignal(matchID, data string) {
    nodeID, endpoint := lookupMatchNode(matchID)

    if nodeID == myNodeID {
        nk.MatchSignal(matchID, data)
    } else {
        httpPost(endpoint + "/rpc/control_match_signal", payload)
    }
}
```

<div class="bilingual">
  <p class="en">The caller does not need to care about locality. The routing layer decides. This is exactly why All-in-One helps so much: when the hot path stays local, the logical architecture can stay layered without paying the full network cost every time.</p>
  <p class="ko">호출하는 쪽은 로컬인지 리모트인지 신경 쓸 필요가 없다. 라우팅 계층이 결정한다. 그리고 바로 이 점 때문에 All-in-One이 크게 유리하다. 핫패스가 로컬에 머물면, 논리적 계층 분리는 유지하면서도 매번 네트워크 비용을 치르지 않아도 된다.</p>
</div>

<div class="bilingual">
  <h3 class="en">Heat Score and dynamic placement</h3>
  <h3 class="ko">Heat Score와 동적 배치</h3>
</div>

<div class="bilingual">
  <p class="en">Not every map is equally hot. High-traffic towns deserve different placement logic from quiet edge maps. So each map carries a <strong>Heat Score</strong>. Hot maps are assigned to the node with the most spare room. Cooler maps are often created locally to save the extra routing hop.</p>
  <p class="ko">모든 맵의 인기도는 같지 않다. 사람들이 몰리는 마을과 외곽 맵은 배치 전략이 달라야 한다. 그래서 각 맵에 <strong>Heat Score</strong>를 준다. 점수가 높은 맵은 가장 여유 있는 노드에 배치하고, 한적한 맵은 추가 라우팅 비용을 아끼기 위해 요청을 받은 노드에서 바로 만드는 경우가 많다.</p>
</div>

<div class="bilingual">
  <h3 class="en">Scaling path</h3>
  <h3 class="ko">확장 경로</h3>
</div>

<div class="bilingual">
  <p class="en">The current four-node setup is basically horizontal scaling plus load balancing. The next step, if it ever becomes necessary, is role-based scaling: CPU-bound GAME nodes and I/O-bound WORLD plus CONTROL nodes. The important part is that the code is already prepared for that transition through <code>ROLE_PROFILE</code>.</p>
  <p class="ko">현재 4노드 All-in-One은 사실상 수평 확장 + 로드밸런싱 구조다. 다음 단계가 필요해진다면 CPU 바운드인 GAME 그룹과 I/O 바운드인 WORLD+CONTROL 그룹을 나누는 역할별 스케일링으로 갈 수 있다. 중요한 건, <code>ROLE_PROFILE</code> 덕분에 코드 차원에서는 이미 그 전환 경로가 준비되어 있다는 점이다.</p>
</div>

```text
Today:
  LB -> Node 1 [ALL]
     -> Node 2 [ALL]
     -> Node 3 [ALL]
     -> Node 4 [ALL]

Future:
  GAME group       -> CPU-oriented scaling
  WORLD+CONTROL    -> connection / I/O-oriented scaling
```

<div class="bilingual">
  <h3 class="en">Connection pool budget</h3>
  <h3 class="ko">커넥션 풀 설계</h3>
</div>

```text
PostgreSQL max_connections = 2000
4 nodes x 350 max_open per node = 1400
Reserve = 600
```

<div class="bilingual">
  <p class="en">I intentionally stop at around 70 percent. If I spend 100 percent of the connection budget on runtime traffic, I leave nothing for migrations, monitoring, or emergency access. Operations always need reserve capacity.</p>
  <p class="ko">의도적으로 70% 정도에서 멈춘다. 런타임 트래픽에 커넥션 예산 100%를 다 써버리면 마이그레이션, 모니터링, 긴급 점검을 위한 여유가 남지 않는다. 운영에는 항상 예비 용량이 필요하다.</p>
</div>

<div class="bilingual">
  <h3 class="en">HTTP key consistency</h3>
  <h3 class="ko">HTTP_KEY 정합성</h3>
</div>

<div class="bilingual">
  <p class="en">One of the dumbest real failure modes in a cluster is still secret mismatch. If node A and node B disagree on the internal HTTP key, cross-node GAME -> WORLD RPC starts failing with unauthorized responses. The process is alive, but items do not get picked up. So I validate this explicitly in CI with <code>scripts/check_config_projection_sync.sh</code>.</p>
  <p class="ko">클러스터에서 실제로 가장 허망한 장애 원인 중 하나는 여전히 시크릿 불일치다. 노드 A와 노드 B의 내부 HTTP_KEY가 다르면, cross-node GAME → WORLD RPC가 인증 실패로 떨어진다. 프로세스는 살아 있는데 아이템이 안 주워지는 식의 장애가 난다. 그래서 <code>scripts/check_config_projection_sync.sh</code>로 CI에서 명시적으로 검증한다.</p>
</div>

> **Core lesson of this section:** I split layers logically early, but I only split them physically when the benefit clearly beats the added operating cost.

---

<div class="bilingual">
  <h2 class="en">Consistency guarantees: what I protect and what I give up</h2>
  <h2 class="ko">일관성 보장: 어디까지 보장하고 어디서 포기하는가</h2>
</div>

<div class="bilingual">
  <h3 class="en">Authority matrix</h3>
  <h3 class="ko">Authority 매트릭스</h3>
</div>

<div class="table-wrap bilingual">
  <table class="en">
    <thead>
      <tr>
        <th>State</th>
        <th>Authority</th>
        <th>Cache</th>
        <th>Recovery if lost</th>
      </tr>
    </thead>
    <tbody>
      <tr>
        <td>Inventory</td>
        <td>WORLD DB</td>
        <td>GAME cache</td>
        <td>Reload from DB</td>
      </tr>
      <tr>
        <td>Gold / XP</td>
        <td>WORLD DB</td>
        <td>GAME HUD cache</td>
        <td>Reload from DB</td>
      </tr>
      <tr>
        <td>Saved position</td>
        <td>WORLD DB</td>
        <td>None</td>
        <td>DB is the source of truth</td>
      </tr>
      <tr>
        <td>Live position</td>
        <td>GAME memory</td>
        <td>Client</td>
        <td>Return to last saved position</td>
      </tr>
      <tr>
        <td>Monster / NPC state</td>
        <td>GAME memory</td>
        <td>None</td>
        <td>Disappear and respawn</td>
      </tr>
      <tr>
        <td>Trade session state</td>
        <td>GAME memory</td>
        <td>Both clients</td>
        <td>Timeout and cancel</td>
      </tr>
      <tr>
        <td>Party</td>
        <td>Redis (Lua atomic)</td>
        <td>GAME cache</td>
        <td>Redis snapshot</td>
      </tr>
    </tbody>
  </table>

  <table class="ko">
    <thead>
      <tr>
        <th>상태</th>
        <th>Authority</th>
        <th>캐시</th>
        <th>유실 시 복구</th>
      </tr>
    </thead>
    <tbody>
      <tr>
        <td>인벤토리</td>
        <td>WORLD DB</td>
        <td>GAME 캐시</td>
        <td>DB에서 재로드</td>
      </tr>
      <tr>
        <td>골드 / 경험치</td>
        <td>WORLD DB</td>
        <td>GAME HUD 캐시</td>
        <td>DB에서 재로드</td>
      </tr>
      <tr>
        <td>저장된 위치</td>
        <td>WORLD DB</td>
        <td>없음</td>
        <td>DB 정본</td>
      </tr>
      <tr>
        <td>실시간 위치</td>
        <td>GAME 메모리</td>
        <td>클라이언트</td>
        <td>마지막 저장 위치로 복귀</td>
      </tr>
      <tr>
        <td>몬스터 / NPC 상태</td>
        <td>GAME 메모리</td>
        <td>없음</td>
        <td>증발 후 리스폰</td>
      </tr>
      <tr>
        <td>거래 진행 상태</td>
        <td>GAME 메모리</td>
        <td>양쪽 클라이언트</td>
        <td>타임아웃 후 취소</td>
      </tr>
      <tr>
        <td>파티</td>
        <td>Redis (Lua 원자)</td>
        <td>GAME 캐시</td>
        <td>Redis 스냅샷</td>
      </tr>
    </tbody>
  </table>
</div>

<div class="bilingual">
  <h3 class="en">What I gave up on purpose</h3>
  <h3 class="ko">의식적으로 포기한 것들</h3>
</div>

<div class="bilingual">
  <p class="en"><strong>I gave up durable live-position tracking.</strong> Player position lives in GAME memory. If the server dies suddenly, the player returns to the last saved position, usually up to about thirty seconds behind. That sounds ugly, but the alternative would be position writes at gameplay frequency. At 20Hz, that is impossible to justify against inventory and economy writes.</p>
  <p class="ko"><strong>실시간 위치의 영속성은 포기했다.</strong> 플레이어 위치는 GAME 메모리에 있다. 서버가 갑자기 죽으면 보통 마지막 저장 시점, 대략 30초 전 위치로 돌아간다. 보기에는 아쉽지만, 대안은 게임 플레이 빈도로 위치를 DB에 쓰는 것이다. 20Hz 위치 쓰기를 인벤토리나 경제 쓰기보다 우선시할 수는 없다.</p>
</div>

<div class="bilingual">
  <p class="en"><strong>I gave up persistent monster and NPC state.</strong> If the node dies, they disappear and respawn. That is acceptable because monster lifecycles are already inherently disposable.</p>
  <p class="ko"><strong>몬스터와 NPC 상태의 영속성도 포기했다.</strong> 노드가 죽으면 이들은 사라지고 다시 리스폰된다. 몬스터의 수명 자체가 원래 일시적이기 때문에 이건 감당 가능한 손실이다.</p>
</div>

<div class="bilingual">
  <p class="en"><strong>I accept session-bound trade cancellation.</strong> If one participant disconnects, the trade is cancelled. The rule is blunt, but it is predictable and safe.</p>
  <p class="ko"><strong>거래의 세션 의존성도 수용했다.</strong> 한 쪽이 접속을 끊으면 거래는 취소된다. 거칠지만 예측 가능하고 안전한 규칙이다.</p>
</div>

<div class="bilingual">
  <h3 class="en">Payload caching</h3>
  <h3 class="ko">캐시 전략</h3>
</div>

<div class="bilingual">
  <p class="en">Broadcasting the same inventory or status state to many nearby players should not require reserializing the same protobuf fifty times. So the player object keeps cached payloads that are invalidated only when the underlying state changes.</p>
  <p class="ko">근처 50명에게 같은 인벤토리나 상태를 보낼 때, 같은 protobuf를 50번 다시 직렬화할 필요는 없다. 그래서 플레이어 객체에 캐시된 payload를 두고, 실제 상태가 바뀌었을 때만 무효화한다.</p>
</div>

```go
type Player struct {
    CachedInvSyncPayload []byte
    CachedStatusPayload  []byte
}
```

<div class="bilingual">
  <p class="en">This is the same basic idea as fragment caching or ETag-driven reuse on the web. If nothing changed, do not rebuild the payload.</p>
  <p class="ko">이건 웹의 fragment cache나 ETag 기반 재사용과 같은 원리다. 바뀐 게 없으면 다시 만들지 않는다.</p>
</div>

> **Core lesson of this section:** consistency is not a moral absolute. It is a budget. You need to decide where it is worth spending.

---

<div class="bilingual">
  <h2 class="en">Presence GC: ghost sessions at 10K+ CCU</h2>
  <h2 class="ko">프레즌스 GC: 10K+ CCU에서의 유령 세션</h2>
</div>

<div class="bilingual">
  <h3 class="en">Ten thousand users disconnected at once, and <code>MatchLeave</code> did not always fire</h3>
  <h3 class="ko">10,000명이 동시에 나갔다. MatchLeave가 호출되지 않았다.</h3>
</div>

<div class="bilingual">
  <p class="en">In theory, Nakama presence should call <code>MatchLeave</code> when a player disconnects. At scale, theory bends. If ten thousand users vanish at once because of maintenance or a network event, the presence event queue can saturate. Some leave events never arrive where I need them.</p>
  <p class="ko">이론대로라면 플레이어가 나가면 Nakama 프레즌스 시스템이 <code>MatchLeave</code>를 호출해줘야 한다. 하지만 규모가 커지면 이론이 휘어진다. 점검이나 네트워크 장애로 10,000명이 동시에 끊기면 프레즌스 이벤트 큐가 포화되고, 필요한 위치까지 도달하지 못하는 leave 이벤트가 생긴다.</p>
</div>

<div class="bilingual">
  <p class="en">That leaves ghost players behind: the match still thinks they are present, metrics lie, and legitimate new players get blocked because the map appears fuller than it is.</p>
  <p class="ko">그러면 서버에는 유령 플레이어가 남는다. 매치는 아직 접속 중이라고 생각하고, 메트릭은 틀어지고, 맵이 실제보다 꽉 찬 것처럼 보여서 정상 유저의 입장이 막히기도 한다.</p>
</div>

<div class="bilingual">
  <h3 class="en">Resolution</h3>
  <h3 class="ko">해결</h3>
</div>

<div class="bilingual">
  <p class="en">I periodically compare Nakama's real presence count against the local player map.</p>
  <p class="ko">주기적으로 Nakama의 실제 프레즌스 수와 GAME의 로컬 플레이어 맵을 비교한다.</p>
</div>

```text
nakamaSize = presence count reported by nk.MatchGet()
localSize  = players tracked in matchState.Players

if nakamaSize == 0:
    clean up every local player as a ghost

if nakamaSize < localSize:
    sync metrics only
    do not eagerly delete specific players
```

<div class="bilingual">
  <p class="en">Why not just delete players when <code>nakamaSize &lt; localSize</code>? Because delayed <code>MatchLeave</code> may still arrive later. If I pre-delete the player and the framework later sends the leave callback, I can create nil-pointer paths or double-cleanup bugs. Only the zero-presence case is safe enough to clean aggressively.</p>
  <p class="ko">왜 <code>nakamaSize &lt; localSize</code>일 때 바로 정리하지 않느냐면, 지연된 <code>MatchLeave</code>가 나중에 도착할 수 있기 때문이다. 내가 먼저 지워버리면 프레임워크가 뒤늦게 leave를 보낼 때 nil 포인터나 이중 정리 버그가 생길 수 있다. 프레즌스가 완전히 0인 경우만 적극적으로 정리해도 안전하다.</p>
</div>

> **Core lesson of this section:** framework guarantees are not infinite. At some scale, you need repair logic around them.

---

<div class="bilingual">
  <h2 class="en">The match loop: an eight-stage pipeline</h2>
  <h2 class="ko">매치 루프: 8단계 파이프라인</h2>
</div>

<div class="bilingual">
  <p class="en">This is where all the earlier decisions converge. The match loop runs at 20Hz, which means 50ms per tick. Everything has to fit into that budget.</p>
  <p class="ko">지금까지의 결정이 전부 모이는 곳이 매치 루프다. 매치 루프는 20Hz, 즉 틱당 50ms로 돈다. 모든 일이 이 예산 안에 들어와야 한다.</p>
</div>

```text
Stage 1: state reset + ghost / eviction guard
Stage 2: lazy join follow-up (max 200 per tick)
Stage 3: deferred RPC result drain
Stage 4: party event apply
Stage 5: message handling
Stage 6: NPC / monster tick (parallel)
Stage 7: AOI update + movement delta detection
Stage 8: batch flush + autosave + heartbeat
```

<div class="bilingual">
  <p class="en">The stages map directly onto the rest of the architecture. Lazy Join lives in Stage 2. Async WorldRPC result handling lives in Stage 3. NPC and monster logic land in Stage 6. AOI filtering and cached payload flush happen at the back of the pipeline. The point is not just order. The point is keeping unrelated work from colliding in the same moment.</p>
  <p class="ko">이 단계들은 앞에서 설명한 아키텍처와 직접 연결된다. Lazy Join은 Stage 2에 있고, Async WorldRPC 결과 수거는 Stage 3에 있고, NPC/몬스터 로직은 Stage 6에 있다. AOI 필터링과 캐시된 payload flush는 마지막 단계에 간다. 중요한 건 순서 자체보다, 서로 다른 종류의 작업이 같은 순간에 뒤엉키지 않게 만드는 것이다.</p>
</div>

<div class="bilingual">
  <h3 class="en">A concrete optimization example</h3>
  <h3 class="ko">파이프라인 최적화의 실제 사례</h3>
</div>

<div class="bilingual">
  <p class="en">Stage 1 used to include a full session refresh every thirty seconds. In a 3K CCU match, that meant touching all 3,000 player sessions in the same tick. That one operation consumed 17.8ms, more than a third of the entire tick budget.</p>
  <p class="ko">Stage 1에는 30초마다 모든 플레이어 세션을 갱신하는 로직이 있었다. 3K CCU 매치에서는 그 말이 곧 3,000명 세션을 한 틱에 전부 터치한다는 뜻이었다. 그 한 작업이 17.8ms를 먹었고, 틱 예산의 3분의 1 이상을 날려버렸다.</p>
</div>

<div class="bilingual">
  <p class="en">The fix was not "make it faster" in the micro sense. The fix was <strong>spread the work across 300 ticks</strong> with hash bucketing, so only about ten users are touched per tick. The cost dropped from 17.8ms to 2.35ms, a 7.6x improvement.</p>
  <p class="ko">해결은 미시적으로 "더 빠르게"가 아니었다. <strong>해시 버케팅으로 300틱에 걸쳐 분산</strong>해서 틱당 10명 정도만 터치하게 만들었다. 비용은 17.8ms에서 2.35ms로 내려갔다. 7.6배 개선이다.</p>
</div>

<div class="bilingual">
  <p class="en">That was an important reminder: "once every thirty seconds" can still be expensive if all the work lands in the same tick.</p>
  <p class="ko">여기서 중요한 교훈이 있었다. "30초에 한 번"이라는 말은 안전을 보장하지 않는다. 그 일이 한 틱에 몰리면 여전히 비싸다.</p>
</div>

<div class="bilingual">
  <h3 class="en">Stable-state numbers</h3>
  <h3 class="ko">3K CCU 안정 지표</h3>
</div>

```text
3,000 CCU single-map stable run
  total requests: 2,577,056
  success rate:   100%
  TPS:            4,295
  movement p50:   19μs
  movement p95:   407μs
  movement p99:   1.83ms
```

<div class="bilingual">
  <p class="en">Those numbers came only after the pipeline was refined under failure. Before those fixes, the same codebase was generating massive reconnect storms under much lower stress. The order and shape of work inside the loop mattered as much as any macro architecture choice.</p>
  <p class="ko">이 수치는 실패를 겪으면서 파이프라인을 정제한 뒤에 나온 값이다. 그 전에는 같은 코드베이스가 훨씬 낮은 부하에서도 대규모 재접속 폭풍을 만들고 있었다. 매크로 아키텍처만큼이나 루프 안에서 일이 배치되는 순서와 형태가 중요했다.</p>
</div>

> **Core lesson of this section:** in a 50ms tick budget, there is no such thing as a "small" task if you accidentally do it 3,000 times in the same frame.

---

<div class="bilingual">
  <h2 class="en">Looking back</h2>
  <h2 class="ko">회고: 돌이켜보면</h2>
</div>

<div class="bilingual">
  <h3 class="en">What turned out to be the right call</h3>
  <h3 class="ko">잘한 결정</h3>
</div>

<div class="bilingual">
  <p class="en"><strong>Validating the "map equals match" hack early</strong> was decisive. If that prototype had failed, the framework choice itself would have been wrong. The fact that the prototype worked gave the rest of the architecture permission to exist.</p>
  <p class="ko"><strong>"맵 = 매치" 꼼수를 일찍 검증한 것</strong>이 결정적이었다. 그 프로토타입이 안 됐다면 프레임워크 선택 자체가 틀렸을 것이다. 그게 성립한다는 걸 초기에 확인했기 때문에 나머지 아키텍처가 가능해졌다.</p>
</div>

<div class="bilingual">
  <p class="en"><strong>The GAME / WORLD / CONTROL split</strong> ended up being the most valuable long-term decision. Once the authority lines were clear, debugging also became clearer. Inventory bugs point toward WORLD. Lag points toward GAME. Admission failures point toward CONTROL.</p>
  <p class="ko"><strong>3계층 분리(GAME / WORLD / CONTROL)</strong>는 장기적으로 가장 값진 결정이었다. 권위선이 명확해지니까 디버깅도 같이 명확해졌다. 인벤토리 문제는 WORLD, 렉은 GAME, 접속 불가는 CONTROL을 먼저 보면 된다.</p>
</div>

<div class="bilingual">
  <p class="en"><strong>Choosing authority first</strong> became a durable habit. Before adding a feature, I ask where truth lives. That question prevents a surprising number of future bugs.</p>
  <p class="ko"><strong>Authority를 먼저 정하는 습관</strong>도 크게 남았다. 기능을 추가하기 전에 "이 상태의 진실은 어디에 있는가?"를 묻는 습관이 의외로 많은 미래 버그를 막아준다.</p>
</div>

<div class="bilingual">
  <p class="en"><strong>Building Queue Admission early</strong> was another good call. Safety belts are not installed after the crash. They are installed before it.</p>
  <p class="ko"><strong>Queue Admission을 일찍 만든 것</strong>도 잘한 결정이었다. 안전벨트는 사고 난 뒤에 매는 게 아니라, 그 전에 매야 한다.</p>
</div>

<div class="bilingual">
  <h3 class="en">What cost more than expected</h3>
  <h3 class="ko">비용을 치른 결정</h3>
</div>

<div class="bilingual">
  <p class="en"><strong>WorldRPC lane classification has a learning cost.</strong> Every new feature forces the question: is this Query, Sync, Commit, or Async? A wrong answer can mean either missing idempotency or adding needless latency.</p>
  <p class="ko"><strong>WorldRPC 4차선 추상화는 학습 비용이 있다.</strong> 새 기능을 만들 때마다 이 RPC가 Query인지, Sync인지, Commit인지, Async인지 판단해야 한다. 잘못 분류하면 멱등성이 빠지거나, 불필요하게 느려진다.</p>
</div>

<div class="bilingual">
  <p class="en"><strong>Lua DSL debugging is undeniably worse than Go debugging.</strong> But I still think the content separation is worth it.</p>
  <p class="ko"><strong>Lua DSL의 디버깅 경험은 확실히 Go보다 나쁘다.</strong> 그래도 콘텐츠 분리의 가치가 더 크다고 본다.</p>
</div>

<div class="bilingual">
  <p class="en"><strong>All-in-One means resource contention remains real.</strong> CPU spikes in GAME can still bleed into WORLD's I/O behavior. I consider that acceptable for now, but I expect more architectural changes somewhere around the 15K-20K range if traffic keeps growing.</p>
  <p class="ko"><strong>All-in-One은 자원 경합을 완전히 없애지 못한다.</strong> GAME의 CPU 스파이크가 WORLD의 I/O 처리에도 영향을 줄 수 있다. 지금은 감당 가능한 수준이지만, 트래픽이 계속 커지면 15K~20K 정도 구간에서 추가 구조 변경이 필요하리라 예상한다.</p>
</div>

<div class="bilingual">
  <h3 class="en">What I got wrong and changed</h3>
  <h3 class="ko">틀렸고, 바꿨다</h3>
</div>

<div class="bilingual">
  <p class="en"><strong>I touched every session in one tick.</strong> "Once every thirty seconds" sounded light. At 3,000 players, it cost 17.8ms in a single frame. The real lesson was that burst size matters more than calendar frequency.</p>
  <p class="ko"><strong>세션 터치를 한 틱에 몰아서 했다.</strong> "30초에 한 번"은 가벼워 보였다. 하지만 3,000명을 한 프레임에 다 터치하면 17.8ms였다. 핵심 교훈은 빈도보다도 한 번에 몰리는 양이 더 중요하다는 점이었다.</p>
</div>

<div class="bilingual">
  <p class="en"><strong>I thought a five-second heartbeat would be enough.</strong> It was slow enough to let the cluster lie to itself during reconnect storms.</p>
  <p class="ko"><strong>하트비트 5초면 충분할 거라고 생각했다.</strong> 실제로는 재접속 폭풍 동안 클러스터가 자기 자신에게 거짓말을 하게 만들 만큼 느렸다.</p>
</div>

<div class="bilingual">
  <p class="en"><strong>I let NPC ticks call WORLD directly.</strong> That made the content layer capable of DOSing the persistence layer.</p>
  <p class="ko"><strong>NPC 틱에서 WORLD를 직접 부르게 했다.</strong> 그 순간 콘텐츠 레이어가 영속 계층을 공격할 수 있는 구조가 되었다.</p>
</div>

<div class="bilingual">
  <p class="en">All three mistakes shared one pattern: something that felt "probably fine" at design time was only revealed as false under stress.</p>
  <p class="ko">세 가지 실수의 공통점은 하나다. 설계 시점에는 "아마 괜찮겠지"라고 생각했던 것이, 스트레스 테스트를 걸어보니 아니었다.</p>
</div>

<div class="bilingual">
  <h3 class="en">What I intentionally did not split</h3>
  <h3 class="ko">의식적으로 안 나눈 것들</h3>
</div>

<div class="bilingual">
  <p class="en"><strong>I did not add a Repository pattern.</strong> It would improve certain kinds of test seams, but for this codebase it would also spread one query change across interfaces, implementations, and mocks. Right now, direct SQL inside domain functions is simpler and more legible.</p>
  <p class="ko"><strong>Repository 패턴은 도입하지 않았다.</strong> 테스트 seam은 좋아지겠지만, 지금 규모에서는 쿼리 하나 고칠 때 인터페이스, 구현체, mock까지 같이 고쳐야 한다. 현재는 도메인 함수 안에서 직접 SQL을 실행하는 편이 더 읽기 쉽고 빠르다.</p>
</div>

<div class="bilingual">
  <p class="en"><strong>I did not break CONTROL out into dedicated nodes.</strong> Queue and chat can be isolated later if they ever dominate independently. For now, the cost of operating that separation is larger than the benefit.</p>
  <p class="ko"><strong>CONTROL 전용 노드도 따로 두지 않았다.</strong> 큐와 채팅이 정말 독립적으로 커지면 나눌 수 있겠지만, 지금은 분리 운영 비용이 얻는 이득보다 크다.</p>
</div>

<div class="bilingual">
  <p class="en"><strong>I deliberately limited OpsEngine automation.</strong> Being able to automate a dangerous action does not mean it should be automated.</p>
  <p class="ko"><strong>OpsEngine의 자동 조치 범위도 제한했다.</strong> 자동화할 수 있다는 것과 자동화해야 한다는 것은 다르다.</p>
</div>

<div class="bilingual">
  <h2 class="en">Closing</h2>
  <h2 class="ko">마무리</h2>
</div>

<div class="bilingual">
  <p class="en">The numbers in this post are not estimates pulled from a design document. They came from building the system, stressing it, watching it break, and then repairing it. In single-map chaos tests, the server was very stable until around 2,500 CCU, started breaking around 2,750, and could technically accept up to around 4,000 while remaining highly unstable. Across multiple nodes and multiple maps, I validated roughly 20K-25K CCU. In live service, I would only budget 60-70 percent of that.</p>
  <p class="ko">이 글에 나오는 수치들은 설계 문서에서 나온 추정이 아니다. 설계하고, 구현하고, 부하를 걸어보고, 깨지는 걸 보고, 고친 과정의 산물이다. 단일 맵 카오스 테스트에서는 2,500 CCU까지는 아주 안정적이었고, 2,750부터 무너지기 시작했다. 접속 자체는 4,000까지 받지만 매우 불안정하다. 여러 노드, 여러 맵 기준으로는 20K~25K CCU 정도를 검증했고, 라이브 환경에서는 보수적으로 그 60-70% 수준만 기대한다.</p>
</div>

<div class="bilingual">
  <p class="en">A single theme repeats across every section: <strong>explicitly decide what you are giving up.</strong></p>
  <p class="ko">이 글 전반에서 반복되는 주제는 하나다. <strong>무엇을 포기할 것인가를 명시적으로 정하라.</strong></p>
</div>

<div class="bilingual">
  <div class="en">
    <ul>
      <li>I gave up admission speed to protect the cluster during reconnect storms.</li>
      <li>I gave up durable live-position tracking to preserve match-loop performance.</li>
      <li>I gave up immediate NPC-side persistence so WORLD could survive.</li>
      <li>I gave up full automation so a false positive could not shut the service down.</li>
      <li>I gave up some theoretical scalability to keep the All-in-One model simple.</li>
      <li>I gave up Lua debugging comfort to separate content from the engine.</li>
    </ul>
  </div>
  <div class="ko">
    <ul>
      <li>입장 속도를 포기하고 재접속 폭풍에서 클러스터를 지켰다.</li>
      <li>실시간 위치의 영속성을 포기하고 매치 루프 성능을 얻었다.</li>
      <li>NPC의 즉시 영속 실행을 포기하고 WORLD를 살렸다.</li>
      <li>자동화 범위를 포기하고 오탐 한 번에 서비스가 내려가는 걸 막았다.</li>
      <li>일부 확장성을 포기하고 All-in-One의 단순성을 택했다.</li>
      <li>Lua 디버깅 편의성을 포기하고 콘텐츠와 엔진을 분리했다.</li>
    </ul>
  </div>
</div>

<div class="bilingual">
  <p class="en">A good architecture is not one that does everything well. It is one that can explain what it chose not to do, and why that cost is acceptable.</p>
  <p class="ko">좋은 아키텍처는 모든 걸 잘하는 아키텍처가 아니다. 무엇을 하지 않기로 했는지, 그리고 왜 그 대가가 감당 가능하다고 판단했는지 설명할 수 있는 아키텍처다.</p>
</div>

<div class="bilingual">
  <p class="en">This server is not perfect. But I can explain why it looks like this, and I can explain where it broke.</p>
  <p class="ko">완벽한 서버는 아니지만, 왜 이렇게 만들었는지는 설명할 수 있다. 그리고 어디서 터졌는지도.</p>
</div>
