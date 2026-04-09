---
title: "Why I Built It This Way: Design Decisions Behind a 2D MMORPG Server Architecture (Part 1)"
title_kr: "왜 이렇게 만들었나: 2D MMORPG 서버 아키텍처 설계 의사결정기 (상)"
date: 2026-04-09 12:00:00 +0800
categories:
  - server-engineering
excerpt_en: "A design-decision log for a 2D MMORPG server: why I chose Nakama, why a map became a match, why GAME, WORLD, and CONTROL were split, and what each decision cost."
excerpt_kr: "2D MMORPG 서버를 만들면서 왜 Nakama를 골랐고, 왜 맵을 매치로 돌렸으며, 왜 GAME, WORLD, CONTROL을 분리했는지와 그 대가를 기록한 설계 의사결정기다."
---

<div class="bilingual">
  <p class="en lede">I am building a 2D MMORPG server: a real-time match system written in Go, custom plugins on top of Nakama, PostgreSQL with Citus, Redis, and a Lua scripting layer. This post is a record of why I made certain architecture decisions. Architecture documents usually preserve only the result. I wanted the blog to preserve the path.</p>
  <p class="ko lede">2D MMORPG 서버를 만들고 있다. Go로 작성된 실시간 매치 시스템, Nakama 위에 올린 커스텀 플러그인, PostgreSQL + Citus 분산 DB, Redis, Lua 스크립팅 레이어. 이 글은 서버를 만들기 위해 왜 이런 결정을 내렸는가에 대한 기록이다. 아키텍처 문서에는 결과만 남지만, 블로그에는 과정을 남기고 싶었다.</p>
</div>

<div class="bilingual">
  <p class="en">I will focus on why I chose this stack, which alternatives I discarded, and what each decision cost me. To make it easier for developers who are not used to game servers, I use comparisons from web-server engineering where possible. In the AI era, none of these individual techniques are especially exotic. What still feels unique to me is the blueprint that emerges when those choices are assembled into one system.</p>
  <p class="ko">이 글에서는 해당 스택을 선택한 이유, 버린 대안, 그리고 그로 인한 대가를 중심으로 적어보려 한다. 게임 서버에 익숙하지 않은 개발자들도 이해할 수 있도록, 가능한 곳에서는 웹 서버 개발의 비유를 곁들였다. AI 시대에 개별 기술 구현 자체는 그리 특별할 것 없는 조합일지 몰라도, 그 결정들이 하나의 설계도로 조립되는 과정은 여전히 유니크하다고 생각한다.</p>
</div>

<div class="bilingual">
  <h2 class="en">Starting point: why Nakama</h2>
  <h2 class="ko">출발점: 왜 Nakama인가</h2>
</div>

<div class="bilingual">
  <p class="en">There are plenty of game-server framework options: Photon, Mirror, Colyseus, or building everything from scratch. I chose Nakama for three reasons.</p>
  <p class="ko">게임 서버 프레임워크는 선택지가 꽤 있다. Photon, Mirror, Colyseus, 또는 아예 처음부터 짜는 방법. 내가 Nakama를 선택한 이유는 세 가지였다.</p>
</div>

<div class="bilingual">
  <p class="en"><strong>First, I did not have time to rebuild wheels that someone else had already built.</strong> Authentication, session management, matchmaking, and presence tracking can easily consume months before any real game logic even starts. Nakama already ships with those pieces. In web terms, it is closer to choosing Django or Rails than hand-rolling authentication, sessions, and middleware from raw sockets.</p>
  <p class="ko"><strong>첫째, 누군가 이미 해 둔 바퀴를 다시 만들 시간이 없었다.</strong> 인증, 세션 관리, 매치메이킹, 프레즌스(접속 상태 추적) 같은 것들을 하나하나 구현하면 게임 로직을 쓰기도 전에 몇 달이 간다. Nakama는 이걸 다 내장하고 있다. 웹 개발로 치면, 인증/세션/미들웨어를 직접 만드는 대신 Django나 Rails 같은 풀스택 프레임워크를 쓰는 것과 비슷하다.</p>
</div>

<div class="bilingual">
  <p class="en"><strong>Second, the Go runtime plugin model.</strong> Nakama supports three server-side runtimes: Lua, TypeScript, and Go. For a performance-sensitive game server, Lua or TypeScript never felt like the right primary runtime. The Go plugin is built as a <code>.so</code> and loaded into Nakama, which means I still get native Go performance where it matters most. That alone made it a first-class candidate.</p>
  <p class="ko"><strong>둘째, Go 런타임 플러그인.</strong> Nakama는 Lua, TypeScript, Go 세 가지 서버 사이드 런타임을 지원한다. 성능이 중요한 게임 서버에서 Lua나 TypeScript는 주력 런타임으로 적합하지 않았다. Go 플러그인은 <code>.so</code> 파일로 빌드해서 Nakama에 로드하는 방식인데, 네이티브 Go 성능을 그대로 쓸 수 있다. 이 정도면 충분히 최우선 고려 대상이었다.</p>
</div>

<div class="bilingual">
  <p class="en"><strong>Third, it is open source.</strong> Being able to read the framework source means that when I hit a limit, I still have a way to understand the internals and route around it. I have had to inspect Nakama's presence behavior more than once, and each time the ability to read the source turned from a nice-to-have into a practical debugging tool. Even at the licensing level, it landed among the strongest options I could seriously consider.</p>
  <p class="ko"><strong>셋째, 오픈소스.</strong> 소스를 읽을 수 있다는 건 프레임워크의 한계를 만났을 때 내부 동작을 이해하고 우회로를 찾을 수 있다는 뜻이다. 실제로 프레즌스 시스템의 내부 동작을 이해해야 하는 상황이 여러 번 있었고, 그때마다 소스를 읽어서 해결했다. 라이선스까지 고려해도 충분히 좋은 선택지였다.</p>
</div>

<div class="bilingual">
  <p class="en"><strong>The discarded alternative</strong> was building everything from scratch: a raw TCP server, protobuf on top, and self-managed sessions from day one. I considered it seriously. But if a solo project spends six months on infrastructure, the game may never arrive. I decided to accept the fifty percent Nakama could do for me and focus on the other fifty percent that was actually unique to my game.</p>
  <p class="ko"><strong>버린 대안</strong>은 처음부터 직접 짜는 방식이었다. TCP 서버 위에 protobuf를 얹고, 세션 관리부터 전부 만드는 구조다. 진지하게 고민했다. 하지만 혼자 하는 프로젝트에서 인프라에 6개월을 쓰면 게임은 영영 안 나온다. Nakama가 해주는 50%를 받아들이고, 나머지 50%에 집중하기로 했다.</p>
</div>

<div class="bilingual">
  <p class="en"><strong>The cost</strong> is that Nakama's match system is fundamentally <strong>single-goroutine</strong>. <code>MatchLoop</code> runs on one goroutine per tick, and any I/O can block that pipeline. That is not my design choice. It is the framework's constraint. But honestly, if I had designed the loop myself, I probably would have chosen something close to this anyway, so I never saw it as a disastrous tradeoff.</p>
  <p class="ko"><strong>대가</strong>는 Nakama의 매치 시스템이 기본적으로 <strong>싱글 고루틴</strong>이라는 점이다. <code>MatchLoop</code>는 한 틱에 하나의 고루틴에서 돌고, 모든 I/O가 그 파이프라인을 블로킹할 수 있다. 이건 내가 선택한 설계라기보다 프레임워크의 제약이다. 그래도 내가 직접 설계했더라도 비슷하게 만들었을 것 같아서, 아주 큰 트레이드오프라고 보지는 않았다.</p>
</div>

<div class="bilingual">
  <h2 class="en">The biggest hack: map equals match</h2>
  <h2 class="ko">가장 큰 꼼수: 맵 = 매치</h2>
</div>

<div class="bilingual">
  <p class="en">The most important conceptual shift in this architecture is simple to state and strange to hear at first: <strong>one MMORPG map runs as one Nakama match.</strong></p>
  <p class="ko">이 서버 아키텍처에서 가장 핵심적인 발상의 전환은 단순하지만 낯설다. <strong>MMORPG의 맵 하나를 Nakama의 매치 하나로 돌린다.</strong></p>
</div>

<div class="bilingual">
  <h3 class="en">What Nakama means by a match</h3>
  <h3 class="ko">Nakama가 원래 의도한 매치 인스턴스의 개념</h3>
</div>

<div class="bilingual">
  <p class="en">Nakama's match system was designed for <strong>ephemeral sessions</strong>: a battle-royale lobby, a chess match, one round of a card game. Players gather, play, and when the game ends, the match disappears. The lifetime is measured in minutes, maybe an hour.</p>
  <p class="ko">Nakama의 매치 시스템은 <strong>휘발성 세션</strong>을 위해 설계됐다. 배틀로얄 로비, 체스 대국, 카드 게임 한 판처럼. 플레이어들이 모이고, 게임을 하고, 끝나면 매치가 사라진다. 수명은 몇 분에서 길어야 한 시간이다.</p>
</div>

```text
The match lifecycle Nakama assumes:
  create -> players enter -> game runs -> result is decided -> match ends
  (minutes to perhaps one hour)
```

<div class="bilingual">
  <p class="en">So when a match becomes empty, Nakama quite reasonably cleans it up. Why keep an empty chess room alive?</p>
  <p class="ko">매치가 비면 Nakama는 당연히 그 매치를 정리한다. 아무도 없는 체스 방을 왜 살려두겠는가?</p>
</div>

<div class="bilingual">
  <h3 class="en">What I needed instead</h3>
  <h3 class="ko">내가 필요로 했던 것</h3>
</div>

<div class="bilingual">
  <p class="en">A 2D MMORPG map is different. The main village must still exist even when player count drops to zero. NPCs remain there. Monsters keep wandering. Dropped items still exist on the ground. And the map should still be alive not just after one hour, but after one hundred days, as long as the server itself is alive. At 4 a.m., even if no one is around, the village should still exist. When someone logs in the next morning, it should be ready instantly.</p>
  <p class="ko">하지만 2D MMORPG의 맵은 다르다. 사람들이 가장 많이 모이는 마을은 플레이어가 0명이어도 존재해야 한다. NPC가 서 있고, 몬스터가 돌아다니고, 바닥에 떨어진 아이템이 있다. 그리고 매치가 생성된 지 1시간이 지나도, 100일이 지나도, 서버가 살아 있는 한 계속 살아 있어야 한다. 새벽 4시에 아무도 없어도 마을은 유지되어야 하고, 다음 날 아침 누군가 접속하면 즉시 입장할 수 있어야 한다.</p>
</div>

```text
The lifecycle an MMORPG map needs:
  create -> players enter and leave repeatedly -> remain alive indefinitely
  (until the server shuts down)
```

<div class="bilingual">
  <p class="en">The web analogy is this: Nakama matches are closer to HTTP request-response sessions that disappear when the interaction ends. What I needed was closer to a permanently running WebSocket server that keeps working even when no client is attached.</p>
  <p class="ko">웹 개발로 비유하면, Nakama의 매치는 HTTP 요청처럼 끝나면 사라지는 구조에 가깝다. 반면 내가 필요한 것은 클라이언트가 없어도 계속 도는 WebSocket 서버에 더 가깝다.</p>
</div>

<div class="bilingual">
  <h3 class="en">Decision: do not let the match die</h3>
  <h3 class="ko">결정: 매치를 죽이지 않는다</h3>
</div>

<div class="bilingual">
  <p class="en">The core idea is simple: <strong>change the termination rule.</strong></p>
  <p class="ko">핵심 아이디어는 단순하다. <strong>매치의 종료 조건을 바꾼다.</strong></p>
</div>

```text
Default Nakama behavior:
  player count = 0 -> terminate the match

My behavior:
  player count = 0 -> do nothing, keep running the loop
```

<div class="bilingual">
  <p class="en">In simplified form, it looks like this.</p>
  <p class="ko">단순화하면 이런 식이다.</p>
</div>

```go
func checkTermination(st *matchState) bool {
    if st.IsOrphan || st.ForceEvict {
        return true
    }
    if st.StopRequested && len(st.Players) == 0 {
        return true
    }
    return false
}
```

<div class="bilingual">
  <p class="en">As long as <code>StopRequested</code> stays false, the match stays alive forever. Even with zero players, <code>MatchLoop</code> continues at 20 Hz, NPCs keep walking, and monsters keep respawning.</p>
  <p class="ko"><code>StopRequested</code>가 <code>false</code>인 한, 매치는 계속 산다. 플레이어가 0명이어도 <code>MatchLoop</code>는 20Hz로 돌고, NPC는 걷고, 몬스터는 리스폰된다.</p>
</div>

<div class="bilingual">
  <h3 class="en">TTL by map type: villages forever, dungeons for ten minutes</h3>
  <h3 class="ko">맵 유형별 TTL: 마을은 영원히, 던전은 10분</h3>
</div>

<div class="table-wrap bilingual">
  <div class="en">
    <table>
      <thead>
        <tr>
          <th>Type</th>
          <th>Example</th>
          <th>When player count = 0</th>
          <th>GC target?</th>
        </tr>
      </thead>
      <tbody>
        <tr>
          <td><strong>Static</strong></td>
          <td>Village, field</td>
          <td>Keep forever (TTL = -1)</td>
          <td>Never</td>
        </tr>
        <tr>
          <td><strong>OnDemand</strong></td>
          <td>Dynamic dungeon, cave</td>
          <td>Collect after 10 minutes</td>
          <td>Yes</td>
        </tr>
        <tr>
          <td><strong>Personal</strong></td>
          <td>Farm, housing</td>
          <td>Collect after 30 minutes</td>
          <td>Yes</td>
        </tr>
        <tr>
          <td><strong>Procedural</strong></td>
          <td>Procedural dungeon</td>
          <td>Collect after 10 minutes</td>
          <td>Yes</td>
        </tr>
      </tbody>
    </table>
  </div>
  <div class="ko">
    <table>
      <thead>
        <tr>
          <th>유형</th>
          <th>예시</th>
          <th>플레이어 0명 시</th>
          <th>GC 대상</th>
        </tr>
      </thead>
      <tbody>
        <tr>
          <td><strong>Static</strong></td>
          <td>마을, 필드</td>
          <td>영원히 유지 (TTL = -1)</td>
          <td>절대 안 됨</td>
        </tr>
        <tr>
          <td><strong>OnDemand</strong></td>
          <td>동적 던전, 동굴</td>
          <td>10분 후 정리</td>
          <td>됨</td>
        </tr>
        <tr>
          <td><strong>Personal</strong></td>
          <td>개인 농장, 하우징</td>
          <td>30분 후 정리</td>
          <td>됨</td>
        </tr>
        <tr>
          <td><strong>Procedural</strong></td>
          <td>절차적 생성 던전</td>
          <td>10분 후 정리</td>
          <td>됨</td>
        </tr>
      </tbody>
    </table>
  </div>
</div>

<div class="bilingual">
  <p class="en">This is conceptually similar to cache TTL in web services. Some Redis keys live forever, some expire. I ended up treating maps the same way.</p>
  <p class="ko">웹 서비스의 캐시 TTL과 비슷한 개념이다. Redis에서 어떤 키는 영구이고 어떤 키는 TTL이 걸리듯, 맵도 유형에 따라 수명이 다르다.</p>
</div>

<div class="bilingual">
  <h3 class="en">Warm pool: villages are created in advance</h3>
  <h3 class="ko">Warm Pool: 마을은 미리 만들어 놓는다</h3>
</div>

<div class="bilingual">
  <p class="en">Static maps like villages and fields are created ahead of time when the server starts. I call this the <strong>warm pool</strong>. It is the same idea as warming a DB connection pool at process startup so the first request does not pay the cold-start cost.</p>
  <p class="ko">Static 맵인 마을과 필드는 서버 시작 시 미리 만들어 놓는다. 이걸 <strong>Warm Pool</strong>이라고 부른다. 웹 서비스에서 애플리케이션 시작 시 DB 커넥션 풀을 미리 만들어 두는 것과 같은 개념이다. 첫 요청이 cold start를 겪지 않도록 하기 위해서다.</p>
</div>

<div class="bilingual">
  <p class="en">Creating a match can take 100-400 ms once collision-map loading, NPC spawn, and script initialization are included. If that work happens when the player enters, the user sees a loading delay. If it is already warm, entry can be effectively immediate.</p>
  <p class="ko">매치 생성은 충돌 맵 로딩, NPC 스폰, 스크립트 초기화까지 포함하면 100-400ms 정도가 걸린다. 이걸 플레이어 입장 시점에 하면 “맵 로딩 중” 같은 체감 지연이 생긴다. Warm Pool로 미리 만들어 두면 즉시 입장이 가능하다.</p>
</div>

<div class="bilingual">
  <h3 class="en">How four nodes divide fifty maps</h3>
  <h3 class="ko">4대의 노드가 맵을 나눠 갖는 법</h3>
</div>

<div class="bilingual">
  <p class="en">Once the server runs on four nodes, who creates which maps? If all four nodes create all fifty maps, every map exists four times and players scatter across duplicates. The web analogy is the classic “all servers run the same cron job” problem. Only one instance should run it. How do you decide which one?</p>
  <p class="ko">서버가 4대로 늘어나면, 맵 50개를 누가 만들어야 할까. 4대 전부가 50개를 만들면 같은 맵의 매치가 4개씩 생기고 플레이어가 흩어진다. 웹 개발로 비유하면 모든 서버가 동시에 같은 크론잡을 실행하는 문제와 같다. 한 대만 실행해야 한다. 누가 맡을지는 어떻게 정할까?</p>
</div>

<div class="bilingual">
  <p class="en">The answer is deterministic assignment by FNV hash. Hash the map ID, divide by the number of nodes, and the remainder decides ownership.</p>
  <p class="ko">해결책은 FNV 해시를 이용한 결정론적 배분이다. 맵 ID를 해시한 뒤 노드 수로 나눈 나머지가 곧 담당 노드다.</p>
</div>

```go
func isMyWarmPoolMap(mapID string, myIndex, totalNodes int) bool {
    h := fnv.New32a()
    h.Write([]byte(mapID))
    return int(h.Sum32()) % totalNodes == myIndex
}
```

```text
50 maps, 4 nodes:
  hash("village_main") % 4 = 0 -> Node 1 creates it
  hash("forest_east")  % 4 = 2 -> Node 3 creates it
  hash("dungeon_cave") % 4 = 1 -> Node 2 creates it
```

<div class="bilingual">
  <p class="en">Because every node uses the same hash function, the same map list, and the same node order, each node can independently decide “is this mine?” without talking to the others.</p>
  <p class="ko">모든 노드가 같은 해시 함수, 같은 맵 목록, 같은 노드 순서를 갖고 있으니, 별도 통신 없이도 “이 맵을 내가 만들어야 하는가?”를 독립적으로 판단할 수 있다.</p>
</div>

<div class="bilingual">
  <p class="en">That is only the steady-state story, though. If Node 2 dies, all maps owned by Node 2 disappear. So I also run a <strong>Warm Pool Reconciler</strong>, a background worker that periodically detects maps that should exist but do not, then lets one of the surviving nodes recreate them. The hash assignment provides balanced distribution in normal conditions. The reconciler is the failure-time fallback.</p>
  <p class="ko">물론 이것만으로는 부족하다. Node 2가 죽으면 Node 2가 담당하던 맵들이 전부 사라진다. 그래서 <strong>Warm Pool Reconciler</strong>라는 백그라운드 워커가 주기적으로 “있어야 하는데 없는 맵”을 감지하고, 살아 있는 노드 중 하나가 대신 생성한다. 해시 배분은 평상시의 균등 분산을 위한 것이고, Reconciler는 장애 시의 fallback이다.</p>
</div>

<div class="bilingual">
  <p class="en">Some special maps, like a heavily trafficked village, can also be pinned by <code>preferred_node_id</code>. In that case, the explicit pin overrides the hash result.</p>
  <p class="ko">특수한 맵, 예를 들면 인기 마을 같은 경우는 <code>preferred_node_id</code>로 특정 노드에 고정할 수도 있다. 이 경우에는 해시 결과보다 지정 노드가 우선한다.</p>
</div>

<div class="bilingual">
  <h3 class="en">What happens when the player needs a map on another node?</h3>
  <h3 class="ko">플레이어가 다른 노드의 맵에 들어가려면?</h3>
</div>

<div class="bilingual">
  <p class="en">This is the next essential question. If a player is connected to Node 1 but the destination map is running on Node 3, what then? In web terms, this is the familiar microservice situation where Service A gets the request but the actual data or authority lives on Service B.</p>
  <p class="ko">여기서 핵심 질문이 나온다. 플레이어가 Node 1에 접속했는데, 들어가려는 맵이 Node 3에 있으면 어떻게 되는가? 웹 개발로 비유하면, 마이크로서비스 A에 요청이 왔는데 실제 데이터나 권위는 서비스 B에 있는 상황과 비슷하다.</p>
</div>

```text
1. Player asks Node 1 to enter a village
2. Node 1 looks up map_instances:
   "this map is running on Node 3 at node-3:7777"
3. Node 1 replies:
   { endpoint: "node-3:7777", match_id: "abc123" }
4. The client opens a WebSocket to Node 3
5. All real-time gameplay traffic continues directly with Node 3
```

<div class="bilingual">
  <p class="en">This is a redirect pattern, not a proxy. HTTP 302 says “go there instead.” The game server does the same. I avoid proxying because real-time WebSocket traffic does not benefit from an extra hop in the middle.</p>
  <p class="ko">이건 프록시가 아니라 리다이렉트 패턴이다. HTTP 302가 “저 URL로 다시 가라”고 응답하듯, 게임 서버도 “이 노드로 가라”고 응답한다. 프록시를 쓰지 않는 이유는 실시간 WebSocket 통신에 중간 홉을 하나 더 넣고 싶지 않기 때문이다.</p>
</div>

<div class="bilingual">
  <p class="en">To make route lookup fast, I keep a Redis routing cache.</p>
  <p class="ko">이 라우팅 정보를 빠르게 찾기 위해 Redis에 라우팅 캐시를 둔다.</p>
</div>

```json
{
  "endpoint": "node-3:7777",
  "match_id": "abc123",
  "node_id": "node-3",
  "player_count": 42
}
```

<div class="bilingual">
  <p class="en">The cache prevents a PostgreSQL read on every routing decision. If the cache turns out to be stale because a node died, I fall back to PostgreSQL and rebuild from there. This is not very different from DNS result caching or service-discovery cache in web infrastructure.</p>
  <p class="ko">매번 PostgreSQL을 조회하면 느리니까 Redis에 캐시해 두고 빠르게 라우팅한다. 캐시가 틀렸다면, 예를 들어 노드가 죽어서 매치가 사라졌다면, PostgreSQL에서 다시 읽고 캐시를 갱신한다. 웹 인프라에서 DNS 결과를 캐시하거나 서비스 디스커버리 결과를 캐시하는 것과 비슷하다.</p>
</div>

<div class="bilingual">
  <h3 class="en">Singleton guarantee in the cluster</h3>
  <h3 class="ko">클러스터에서의 싱글톤 보장</h3>
</div>

<div class="bilingual">
  <p class="en">If two copies of the same village map exist in a four-node cluster, players can end up in separate copies of what should be one shared world. That is fatal in an MMORPG. The fix is a PostgreSQL <strong>UNIQUE constraint</strong> that acts like a distributed lock.</p>
  <p class="ko">4노드 클러스터에서 같은 마을의 매치가 2개 생기면, 플레이어들이 같은 마을이어야 할 공간의 서로 다른 복사본에 들어가 버릴 수 있다. MMORPG에서는 치명적이다. 해결책은 PostgreSQL의 <strong>UNIQUE 제약조건</strong>으로 싱글톤을 보장하는 것이다.</p>
</div>

```sql
CREATE UNIQUE INDEX ON map_instances (map_id, shard_id)
  WHERE status = 'active';
```

<div class="bilingual">
  <p class="en">This is the same idea as distributed locking through <code>SETNX</code> in Redis or through a uniqueness constraint in a shared database. When two nodes race to create the same active map instance, one wins and the other gets a conflict.</p>
  <p class="ko">이건 여러 서버가 동시에 같은 자원을 만들려고 할 때 DB의 UNIQUE 제약으로 하나만 이기게 만드는 것과 같다. Redis의 <code>SETNX</code>로 분산 락을 거는 것과 비슷한 원리다.</p>
</div>

```text
Node 1: INSERT map_instances(map_id='town_center') -> success
Node 2: INSERT map_instances(map_id='town_center') -> conflict -> use Node 1's instance
```

<div class="bilingual">
  <p class="en">Even then, timing weirdness can still create a duplicate briefly. So at <code>MatchInit</code> time, each match verifies whether it is the canonical one. If not, it marks itself orphaned and exits immediately.</p>
  <p class="ko">그래도 타이밍 이슈로 잠깐 중복 매치가 생길 수 있다. 그래서 <code>MatchInit</code> 시점에 “내가 정본(canonical) 매치인가?”를 다시 확인한다. 아니면 고아 매치로 표시하고 즉시 종료한다.</p>
</div>

```go
registered := control.LookupActiveMapInstance(db, mapID)
if registered.MatchID != myMatchID {
    state.IsOrphan = true
    state.StopRequested = true
    return state
}
```

<div class="bilingual">
  <h3 class="en">Heartbeat: proof that a match is still alive</h3>
  <h3 class="ko">하트비트: 매치가 살아있다는 증거</h3>
</div>

<div class="bilingual">
  <p class="en">A match can live forever, but the node hosting it can still die. How do the other nodes know? By heartbeat.</p>
  <p class="ko">매치가 영원히 살아있다고 해도, 그걸 돌리던 노드는 죽을 수 있다. 다른 노드들은 그걸 어떻게 아는가? 하트비트다.</p>
</div>

```text
Every tick:
  last_heartbeat_at = now
  player_count = N

GC loop every 60s:
  if now - last_heartbeat_at > 30s:
    this match is dead -> clean DB state -> recreate it
```

<div class="bilingual">
  <p class="en">This plays the same role as health checks in web infrastructure. A load balancer repeatedly asks an instance if it is healthy. If the answer stops coming, the instance is removed from the pool. The match heartbeat is the world-server version of that pattern.</p>
  <p class="ko">웹 서비스의 헬스체크와 같은 역할이다. 로드밸런서가 인스턴스를 주기적으로 찔러서, 응답이 없으면 풀에서 빼는 것. 매치의 하트비트가 바로 그 역할이다.</p>
</div>

<div class="bilingual">
  <h3 class="en">Panic defense: what if the match keeps dying immediately?</h3>
  <h3 class="ko">패닉 방어: 매치가 계속 죽으면?</h3>
</div>

<div class="bilingual">
  <p class="en">Imagine a map-loading script bug that crashes a certain village right after creation. The GC loop would see the map die, recreate it, watch it panic again, and repeat forever. So I record early death as a panic signal and temporarily block regeneration.</p>
  <p class="ko">맵 로딩 스크립트에 버그가 있어서 특정 마을 매치가 생성되자마자 패닉으로 죽는다고 해보자. 그러면 GC가 “죽었네” → 새로 만들기 → 또 죽음 → 새로 만들기 식의 무한 루프에 빠진다. 그래서 조기 사망을 패닉 신호로 기록하고, 일정 시간 재생성을 막는다.</p>
</div>

```go
if tick < 10 && mapID != "" {
    control.RecordMapPanic(mapID)
    // block regeneration for 5 minutes
}
```

<div class="bilingual">
  <p class="en">That is effectively a circuit breaker for map recovery. Web services use the same idea for failing upstream APIs. If the dependency keeps failing, stop hammering it for a while.</p>
  <p class="ko">이건 맵 복구를 위한 circuit breaker와 같은 패턴이다. 웹 서비스에서도 반복 실패하는 외부 API 호출을 잠시 막아두는 것과 같은 생각이다.</p>
</div>

<div class="bilingual">
  <h3 class="en">Why this hack was possible at all</h3>
  <h3 class="ko">이 꼼수가 가능했던 이유</h3>
</div>

<div class="bilingual">
  <p class="en">This works only because Nakama's match abstraction is more flexible than the product story around it. I could fully control <code>MatchLoop</code>, override termination conditions, and attach metadata like <code>map_id</code> to the match label for routing. Nakama was designed around “match equals temporary game session,” but the actual abstraction was generic enough to be repurposed into “match equals persistent world shard.” It is fair to call that a hack. But it was a productive hack, because it let me avoid writing an MMO world server entirely from scratch.</p>
  <p class="ko">이 방식이 성립하는 건 Nakama의 매치 추상화가 의도보다 더 유연했기 때문이다. <code>MatchLoop</code>를 완전히 제어할 수 있고, 종료 조건도 바꿀 수 있으며, 매치 라벨에 <code>map_id</code> 같은 메타데이터를 붙여 라우팅에 활용할 수 있다. Nakama는 “매치 = 일시적 게임 세션”을 염두에 뒀지만, 실제 구현은 “매치 = 영속 월드 샤드”로도 전용할 수 있을 만큼 범용적이었다. 꼼수라고 부르는 게 맞다. 하지만 이 꼼수 덕분에 MMO 월드 서버를 바닥부터 짜지 않아도 됐다.</p>
</div>

<div class="bilingual">
  <h2 class="en">The first major split: GAME, WORLD, CONTROL</h2>
  <h2 class="ko">3계층 분리: GAME / WORLD / CONTROL</h2>
</div>

<div class="bilingual">
  <p class="en">This was the earliest architectural decision and probably the one with the widest downstream effect.</p>
  <p class="ko">서버 아키텍처에서 가장 먼저 내린 결정이고, 가장 큰 영향을 준 결정이다.</p>
</div>

<div class="bilingual">
  <h3 class="en">The problem</h3>
  <h3 class="ko">문제</h3>
</div>

<div class="bilingual">
  <p class="en">A game server has to satisfy two demands that naturally fight each other.</p>
  <p class="ko">게임 서버는 서로 충돌하는 두 요구를 동시에 만족해야 한다.</p>
</div>

<div class="bilingual">
  <div class="en">
    <ol>
      <li><strong>Real-time responsiveness.</strong> Movement, combat, pickups, and NPC actions need to complete fast enough that the player feels immediate feedback, often inside something like a 50 ms budget.</li>
      <li><strong>Persistent consistency.</strong> If the server crashes after an inventory move, the item still has to end up in a correct state. If gold is granted twice, the economy is corrupted.</li>
    </ol>
  </div>
  <div class="ko">
    <ol>
      <li><strong>실시간 응답.</strong> 이동, 전투, 줍기, NPC 행동은 대략 50ms 수준 안에서 처리되어야 체감이 끊기지 않는다.</li>
      <li><strong>영속적 일관성.</strong> 인벤토리에서 아이템을 옮긴 뒤 서버가 죽어도 상태는 맞아야 하고, 골드가 두 번 지급되면 경제가 깨진다.</li>
    </ol>
  </div>
</div>

<div class="bilingual">
  <p class="en">If you optimize purely for real-time simulation, you minimize DB round-trips. If you optimize purely for consistency, every state change wants to live in a transaction. One layer will never do both equally well.</p>
  <p class="ko">실시간 처리에 최적화하면 DB 왕복을 최소화해야 하고, 일관성에 최적화하면 모든 상태 변경이 트랜잭션 안으로 들어가야 한다. 하나의 레이어가 이 둘을 동시에 잘하기는 어렵다.</p>
</div>

<div class="bilingual">
  <p class="en">The web analogy is familiar: Redis handles fast read paths, PostgreSQL handles durable truth. If all reads go to PostgreSQL, the system gets slow. If all writes only live in cache, the truth evaporates. The game server problem is the same, except “fast reads” have been replaced by “real-time simulation.”</p>
  <p class="ko">웹 서버도 비슷한 분리를 한다. Redis는 빠른 읽기 경로를 맡고, PostgreSQL은 영속적 진실을 맡는다. 모든 읽기를 DB로 보내면 느리고, 모든 쓰기를 캐시에서만 처리하면 상태가 날아간다. 게임 서버도 같은 원리다. 다만 “빠른 읽기” 대신 “실시간 시뮬레이션”이라는 더 극단적인 요구가 있는 셈이다.</p>
</div>

<div class="bilingual">
  <h3 class="en">The decision</h3>
  <h3 class="ko">결정</h3>
</div>

<div class="bilingual">
  <p class="en">I split the server into three roles.</p>
  <p class="ko">서버를 세 개의 역할로 나눴다.</p>
</div>

```text
GAME
  Real-time simulation
  Movement, combat, NPC AI, monster ticks
  Match state lives in memory only
  Direct DB writes are forbidden

WORLD
  Persistent authority
  Inventory, gold, experience, character state
  Every change goes through PostgreSQL transactions
  Exposes dozens of RPC endpoints

CONTROL
  Temporary coordination
  Map-instance routing, admission queues, chat relay
  Uses Redis + PostgreSQL
  State is reconstructible if lost
```

<div class="bilingual">
  <p class="en">The key principle is always the same: <strong>who owns the truth?</strong> In web systems, if the cache disagrees with the database, the database wins. I applied the same rule state by state.</p>
  <p class="ko">핵심 원칙은 언제나 같다. <strong>누가 진실(truth)을 소유하는가?</strong> 웹 서비스에서 캐시와 DB가 다르면 DB가 맞는 것처럼, 게임 서버에서도 상태마다 정본을 정해두었다.</p>
</div>

<div class="table-wrap bilingual">
  <div class="en">
    <table>
      <thead>
        <tr>
          <th>State</th>
          <th>Authority</th>
          <th>Why</th>
        </tr>
      </thead>
      <tbody>
        <tr>
          <td>Inventory</td>
          <td>WORLD (PostgreSQL)</td>
          <td>Prevents duplication and loss, requires transactions</td>
        </tr>
        <tr>
          <td>Real-time position</td>
          <td>GAME (memory)</td>
          <td>Needs fast response, cannot wait on DB</td>
        </tr>
        <tr>
          <td>Saved position</td>
          <td>WORLD (PostgreSQL)</td>
          <td>Reconnects must resume from durable state</td>
        </tr>
        <tr>
          <td>Party membership</td>
          <td>Redis (Lua scripts)</td>
          <td>Needs atomic multi-key updates</td>
        </tr>
        <tr>
          <td>Trade progress</td>
          <td>GAME (memory)</td>
          <td>Both players are in one match; disconnect cancels it</td>
        </tr>
        <tr>
          <td>Map instance registry</td>
          <td>CONTROL (PostgreSQL)</td>
          <td>Should persist, but can be rebuilt</td>
        </tr>
        <tr>
          <td>Queue gate state</td>
          <td>CONTROL (Redis)</td>
          <td>Fast shared coordination, resettable if needed</td>
        </tr>
      </tbody>
    </table>
  </div>
  <div class="ko">
    <table>
      <thead>
        <tr>
          <th>상태</th>
          <th>권위(Authority)</th>
          <th>이유</th>
        </tr>
      </thead>
      <tbody>
        <tr>
          <td>인벤토리</td>
          <td>WORLD (PostgreSQL)</td>
          <td>복제/소실 방지. 트랜잭션 필수.</td>
        </tr>
        <tr>
          <td>실시간 위치</td>
          <td>GAME (메모리)</td>
          <td>빠른 응답 필요. DB 왕복 불가.</td>
        </tr>
        <tr>
          <td>저장된 위치</td>
          <td>WORLD (PostgreSQL)</td>
          <td>접속 끊기면 마지막 저장 위치로 복귀.</td>
        </tr>
        <tr>
          <td>파티 멤버십</td>
          <td>Redis (Lua 스크립트)</td>
          <td>원자적 다중 키 업데이트 필요.</td>
        </tr>
        <tr>
          <td>거래 진행 상태</td>
          <td>GAME (메모리)</td>
          <td>양쪽 플레이어가 같은 매치 안. 끊기면 취소.</td>
        </tr>
        <tr>
          <td>맵 인스턴스 목록</td>
          <td>CONTROL (PostgreSQL)</td>
          <td>영속적이되, 유실 시 재구축 가능.</td>
        </tr>
        <tr>
          <td>큐 게이트 상태</td>
          <td>CONTROL (Redis)</td>
          <td>빠른 공유. 유실 시 재설정 가능.</td>
        </tr>
      </tbody>
    </table>
  </div>
</div>

<div class="bilingual">
  <h3 class="en">Why I did not merge them back together</h3>
  <h3 class="ko">왜 통합하지 않았는가</h3>
</div>

<div class="bilingual">
  <p class="en">The obvious question is: if GAME wrote directly to the database, would it not be faster because there is no RPC hop? Yes, it would. But I would lose two things I care about more.</p>
  <p class="ko">자연스러운 질문은 이거다. “GAME에서 직접 DB에 쓰면 RPC 왕복이 없어서 더 빠르지 않나?” 맞다. 더 빠를 수 있다. 하지만 그 대신 두 가지를 잃는다.</p>
</div>

<div class="bilingual">
  <div class="en">
    <ol>
      <li><strong>The consistency point fragments.</strong> In web services, payment logic spread across many controllers is harder to reason about than one <code>PaymentService</code>. Inventory mutation spread across <code>match_inventory.go</code> and <code>match_trade.go</code> is the same kind of problem.</li>
      <li><strong>Auditability gets worse.</strong> If every inventory mutation passes through one World RPC, one line of logging covers everything. If the writes are scattered, I have to find every call site and I will miss one.</li>
    </ol>
  </div>
  <div class="ko">
    <ol>
      <li><strong>일관성 보장점이 흩어진다.</strong> 웹 서비스에서 결제 처리가 여러 컨트롤러에 퍼져 있는 것보다 하나의 <code>PaymentService</code>를 통과하는 편이 훨씬 명확하다. 인벤토리 변경도 마찬가지다.</li>
      <li><strong>감사가 어려워진다.</strong> 모든 인벤토리 변경이 하나의 WORLD 함수로 모이면 로그 한 줄을 추가하면 된다. 분산되면 모든 호출부를 찾아야 하고, 반드시 하나는 놓친다.</li>
    </ol>
  </div>
</div>

<div class="bilingual">
  <p class="en">The cost is latency. Even local GAME to WORLD RPC on the same node costs a few milliseconds, and cross-node calls can land anywhere in the rough 5-100 ms range depending on the path. If a player picks up an item and the match loop waits on WORLD, that wait is real. How I handle that latency comes later.</p>
  <p class="ko">대가는 지연이다. 같은 노드에서의 LocalRPC도 수 밀리초는 걸리고, 다른 노드면 경로에 따라 대략 5-100ms 수준까지 갈 수 있다. 플레이어가 아이템을 줍는 순간 WORLD RPC를 보내고 매치 루프가 기다려야 한다면, 그 지연은 실재한다. 이 문제를 어떻게 다루는지는 뒤에서 다시 나온다.</p>
</div>

<div class="bilingual">
  <h3 class="en">How I enforce the split</h3>
  <h3 class="ko">이 분리를 강제하는 방법</h3>
</div>

<div class="bilingual">
  <p class="en">Rules are not kept just because I wrote them down. That becomes especially obvious when working with AI coding agents. An agent will happily decide, “just this once, direct DB access is quicker.” So I enforce the rule with a hook. <code>scripts/check_game_db_write.sh</code> blocks any SQL write pattern inside GAME code. It is the same idea as a lint rule or pre-commit hook in web development: if the architecture boundary matters, it should fail automatically when broken.</p>
  <p class="ko">규칙은 적어둔다고 지켜지지 않는다. 특히 AI 코딩 에이전트와 함께 일할 때는 더 그렇다. 에이전트는 “이번만 직접 쓰는 게 빠르겠다”고 쉽게 판단한다. 그래서 훅으로 강제한다. <code>scripts/check_game_db_write.sh</code>는 GAME 코드에서 SQL 쓰기를 감지하면 막는다. 웹 서비스에서 lint 규칙이나 pre-commit hook으로 특정 패턴을 차단하는 것과 같은 생각이다. 아키텍처 경계가 중요하다면, 깨졌을 때 자동으로 실패해야 한다.</p>
</div>

<div class="bilingual">
  <h2 class="en">PostgreSQL: why I chose Citus sharding</h2>
  <h2 class="ko">PostgreSQL: 왜 Citus 샤딩인가</h2>
</div>

<div class="bilingual">
  <h3 class="en">The problem</h3>
  <h3 class="ko">문제</h3>
</div>

<div class="bilingual">
  <p class="en">For MMO workloads, the database hot path often comes from inventory. Item movement, equipment changes, trades, drops, and pickups can produce dozens of inventory queries per player per minute. At ten thousand concurrent users, that turns into hundreds of thousands of queries per minute. A single PostgreSQL instance can only go so far by scaling upward.</p>
  <p class="ko">MMO 게임의 DB 부하는 대체로 인벤토리에서 많이 나온다. 아이템 이동, 장비 교체, 거래, 드롭, 획득 같은 동작이 플레이어 한 명당 분당 수십 번의 인벤토리 쿼리를 만든다. 동접 1만이면 분당 수십만 쿼리다. 단일 PostgreSQL로 버티려면 수직 확장밖에 없는데, 그건 천장이 있다.</p>
</div>

<div class="bilingual">
  <p class="en">The e-commerce analogy is straightforward. If order and payment tables become massive, at some point one database host stops being a satisfying answer. Then the real options are better hardware or sharding. Inventory tables eventually hit the same wall.</p>
  <p class="ko">웹 개발의 비유로는 이커머스의 주문/결제 테이블과 같다. 규모가 커지면 DB 한 대로는 한계가 오고, 결국 더 좋은 장비를 쓰거나 샤딩해야 한다. 게임의 인벤토리 테이블도 같은 문제를 겪는다.</p>
</div>

<div class="bilingual">
  <h3 class="en">The decision</h3>
  <h3 class="ko">결정</h3>
</div>

<div class="bilingual">
  <p class="en">I chose Citus. It is a PostgreSQL extension, which means I still get PostgreSQL transactions, indexes, and join behavior while gaining horizontal scale. The shard key is <code>user_id</code>. If all items for one player live on the same shard, then inventory queries for that player remain local.</p>
  <p class="ko">Citus를 선택했다. PostgreSQL의 확장(extension)이라 PostgreSQL의 트랜잭션, 조인, 인덱스를 그대로 유지하면서 수평 확장이 가능하다. 샤딩 키는 <code>user_id</code>다. 한 플레이어의 아이템이 같은 샤드에 모이면, 그 플레이어의 인벤토리 쿼리는 항상 로컬에서 끝난다.</p>
</div>

```sql
SELECT create_distributed_table('item_locations', 'user_id');
SELECT create_distributed_table('item_instances', 'user_id');
SELECT create_distributed_table('item_events',    'user_id');

SELECT create_reference_table('item_templates');
SELECT create_reference_table('stat_types');
```

<div class="bilingual">
  <p class="en">This is the same pattern SaaS systems use when they shard by <code>tenant_id</code>. In my case, <code>user_id</code> plays the same role.</p>
  <p class="ko">이건 SaaS가 <code>tenant_id</code>로 샤딩하는 것과 같은 패턴이다. 게임에서는 <code>user_id</code>가 그 역할을 한다.</p>
</div>

<div class="bilingual">
  <h3 class="en">The strict shard-key rule</h3>
  <h3 class="ko">샤딩 키 필수 규칙</h3>
</div>

<div class="bilingual">
  <p class="en">One rule falls out of this choice and it is non-negotiable.</p>
  <p class="ko">이 결정에서 파생된 가장 중요한 규칙이 하나 있다.</p>
</div>

```sql
-- correct
SELECT * FROM item_locations
WHERE item_uid = $1 AND user_id = $2
FOR UPDATE;

-- wrong
SELECT * FROM item_locations
WHERE item_uid = $1
FOR UPDATE;
```

<div class="bilingual">
  <p class="en">The second query works today in a small or single-node Citus setup. But once worker nodes are added, it stops being “just a lookup” and starts turning into lock acquisition across every shard. Under real concurrency, that is a path to disaster. So the rule is simple: any query against a distributed table must include the shard key in its <code>WHERE</code> clause.</p>
  <p class="ko">두 번째 쿼리는 지금 당장 동작할 수 있다. 단일 노드 Citus에서는 모든 데이터가 로컬이니까. 하지만 워커 노드가 늘어나는 순간, 이건 단순한 조회가 아니라 모든 샤드에 락을 거는 쿼리가 된다. 실제 동시성 하에서는 재앙에 가깝다. 그래서 규칙은 단순하다. 분산 테이블의 쿼리는 반드시 <code>WHERE</code>절에 샤딩 키를 포함해야 한다.</p>
</div>

<div class="bilingual">
  <h3 class="en">Cross-shard transactions: player-to-player trade</h3>
  <h3 class="ko">크로스 샤드 트랜잭션: 다른 샤드에 있는 유저 간 거래</h3>
</div>

<div class="bilingual">
  <p class="en">The hardest case in sharding is trade. If user A and user B live on different shards, and they want to swap items atomically, how do you do it?</p>
  <p class="ko">샤딩에서 가장 까다로운 문제는 거래다. 유저 A와 유저 B가 서로 다른 샤드에 있을 수 있는데, 이 둘의 아이템을 원자적으로 교환하려면 어떻게 해야 할까?</p>
</div>

```text
User A (user_id = 1001) -> Shard 1
User B (user_id = 5042) -> Shard 3

Trade:
  A's sword -> B
  B's shield -> A
```

<div class="bilingual">
  <p class="en">Citus handles this through <strong>two-phase commit</strong>. It is the classic distributed-database protocol.</p>
  <p class="ko">Citus는 이걸 <strong>2PC(Two-Phase Commit)</strong>로 처리한다. 분산 데이터베이스의 고전적인 프로토콜이다.</p>
</div>

```text
Phase 1: PREPARE
  Coordinator -> Shard 1: can you prepare A's side?
  Coordinator -> Shard 3: can you prepare B's side?

Phase 2: COMMIT
  Coordinator -> Shard 1: commit
  Coordinator -> Shard 3: commit
```

<div class="bilingual">
  <p class="en">The web analogy is a distributed transaction across microservices, like payment and inventory both needing to succeed or both needing to roll back. In web systems, you might solve it with Saga or 2PC. Here, the same idea lands at the database layer.</p>
  <p class="ko">웹 개발의 비유로는 결제 서비스와 재고 서비스가 둘 다 성공하거나 둘 다 롤백해야 하는 분산 트랜잭션과 같다. MSA에서 Saga나 2PC를 고민하는 맥락과 비슷하다. 다만 여기서는 그 문제를 데이터베이스 레벨에서 푸는 셈이다.</p>
</div>

<div class="bilingual">
  <p class="en">In simplified form, the trade transaction looks like this.</p>
  <p class="ko">단순화한 거래 트랜잭션은 이런 식이다.</p>
</div>

```sql
BEGIN;

SELECT * FROM item_locations
  WHERE item_uid = 'sword_001' AND user_id = 1001
  FOR UPDATE;

SELECT * FROM item_locations
  WHERE item_uid = 'shield_001' AND user_id = 5042
  FOR UPDATE;

UPDATE item_locations SET user_id = 5042
  WHERE item_uid = 'sword_001' AND user_id = 1001;

UPDATE item_locations SET user_id = 1001
  WHERE item_uid = 'shield_001' AND user_id = 5042;

UPDATE trades SET status = 'completed'
  WHERE trade_id = 'trade_789';

COMMIT;
```

<div class="bilingual">
  <p class="en">Several ugly cases can happen here, and each one needs a defined story.</p>
  <p class="ko">여기서는 여러 가지 예외 케이스가 생길 수 있고, 각각에 대한 처리 전략이 필요하다.</p>
</div>

<div class="bilingual">
  <div class="en">
    <ol>
      <li><strong>One shard fails during PREPARE.</strong> The coordinator aborts the entire transaction and both sides roll back. The client sees a failed trade, but consistency is preserved.</li>
      <li><strong>The coordinator crashes after PREPARE.</strong> Both shards stay in prepared state until the coordinator recovers and resolves the transaction.</li>
      <li><strong>An item has already disappeared.</strong> <code>SELECT ... FOR UPDATE</code> returns no row, so the trade aborts.</li>
      <li><strong>Lock ordering creates deadlock.</strong> PostgreSQL deadlock detection chooses a loser, aborts it, and the game can retry with an idempotency key.</li>
    </ol>
  </div>
  <div class="ko">
    <ol>
      <li><strong>한쪽 샤드가 PREPARE에 실패.</strong> Coordinator가 전체를 ABORT하고 양쪽 모두 롤백한다. 거래는 실패하지만 정합성은 지켜진다.</li>
      <li><strong>PREPARE 후 COMMIT 전에 Coordinator가 크래시.</strong> 양쪽 샤드는 prepared 상태로 대기하고, Coordinator가 복구된 뒤 resolve된다.</li>
      <li><strong>거래 도중 아이템이 이미 사라짐.</strong> <code>SELECT ... FOR UPDATE</code>가 빈 결과를 반환하면서 트랜잭션이 실패한다.</li>
      <li><strong>락 순서 때문에 데드락 발생.</strong> PostgreSQL이 하나를 abort하고, 게임은 멱등성 키로 안전하게 재시도할 수 있다.</li>
    </ol>
  </div>
</div>

<div class="bilingual">
  <p class="en">Cross-shard transactions are expensive. The reason I still accept them is asymmetry.</p>
  <p class="ko">크로스 샤드 트랜잭션은 비싸다. 그럼에도 받아들인 이유는 비대칭성 때문이다.</p>
</div>

```text
Inventory reads and same-user updates:
  tens of thousands per minute
  local shard
  fast

Trade commits and a few party-style cases:
  far rarer
  cross-shard
  slower, but acceptable
```

<div class="bilingual">
  <p class="en">Most traffic stays single-shard because it includes <code>user_id</code>. The expensive path exists, but it is rare enough to justify the overall model. In game terms, this matters even more than in a web shop. A failed e-commerce checkout can show “please retry.” A game trade that half-succeeds means duplicated or vanished items in a live economy, with both players staring at the same screen. Atomicity is not a convenience there. It is economic integrity.</p>
  <p class="ko">대부분의 연산은 <code>user_id</code>를 포함하기 때문에 단일 샤드에서 끝난다. 비싼 크로스 샤드는 존재하지만 충분히 드물다. 게임에서는 이 점이 웹보다 더 중요해진다. 웹 이커머스에서 주문이 실패하면 “다시 시도해주세요”를 보여줄 수 있지만, 게임 거래가 반만 성공하면 아이템 복제나 소실이 된다. 두 플레이어가 같은 화면을 보며 기다리는 실시간 경제에서, 원자성 보장은 편의가 아니라 무결성 자체다.</p>
</div>

<div class="bilingual">
  <h3 class="en">Idempotency keys: protection against duplicate execution</h3>
  <h3 class="ko">멱등성 키: 중복 실행 방지</h3>
</div>

<div class="bilingual">
  <p class="en">Networks are unreliable. A WORLD RPC can succeed while its response disappears. GAME may then retry what was already applied. That is how double gold grants happen. So I added an idempotency table.</p>
  <p class="ko">네트워크는 불안정하다. WORLD RPC는 성공했는데 응답만 유실될 수 있다. 그러면 GAME은 실패로 오해하고 다시 호출한다. 그 결과 골드가 두 번 지급될 수 있다. 그래서 멱등성 키 테이블을 추가했다.</p>
</div>

```sql
INSERT INTO idempotency_keys (op_id, result, created_at)
VALUES ($1, $2, now())
ON CONFLICT (op_id) DO NOTHING;
```

<div class="bilingual">
  <p class="en">This is exactly the same pattern Stripe and other payment APIs use with an <code>Idempotency-Key</code> header.</p>
  <p class="ko">이건 Stripe 같은 결제 API가 <code>Idempotency-Key</code> 헤더로 중복 결제를 막는 것과 같은 패턴이다.</p>
</div>

<div class="bilingual">
  <h3 class="en">Transaction guardrails</h3>
  <h3 class="ko">트랜잭션 안전장치</h3>
</div>

<div class="bilingual">
  <p class="en">All WORLD DB work goes through <code>dbutil.WithTx</code>. The point is not stylistic neatness. The point is that panic or early returns must still roll back cleanly, especially when distributed transaction state can otherwise linger in an ugly half-finished form.</p>
  <p class="ko">WORLD의 모든 DB 작업은 <code>dbutil.WithTx</code>를 통과한다. 이건 스타일을 통일하려는 취향 문제가 아니라, 패닉이나 조기 반환이 생겨도 반드시 롤백되게 하려는 장치다. 특히 분산 트랜잭션은 어설프게 끊기면 더 지저분한 흔적을 남길 수 있다.</p>
</div>

```go
func WithTx(ctx context.Context, txer Txer, opts *sql.TxOptions,
            fn func(*sql.Tx) error) (err error) {
    tx, err := txer.BeginTx(ctx, opts)
    if err != nil { return err }

    defer func() {
        if p := recover(); p != nil {
            _ = tx.Rollback()
            err = fmt.Errorf("panic in transaction: %v", p)
        }
    }()

    if err = fn(tx); err != nil {
        _ = tx.Rollback()
        return
    }
    return tx.Commit()
}
```

<div class="bilingual">
  <h2 class="en">Redis: why I distinguish authority from cache</h2>
  <h2 class="ko">Redis: Authority와 Cache를 구분한 이유</h2>
</div>

<div class="bilingual">
  <p class="en">Many projects adopt Redis casually. The decision that should never be casual is this: what happens if this Redis data disappears?</p>
  <p class="ko">Redis를 가볍게 채택해 쓰는 프로젝트는 많다. 하지만 “이 Redis 데이터가 날아가면 어떻게 되는가?”를 가볍게 보면 안 된다.</p>
</div>

<div class="bilingual">
  <p class="en">The web analogy is easy again. If a session cache disappears, the user re-logs in. Annoying, but survivable. If a work queue disappears, jobs vanish. If a rate-limiter counter disappears, fairness shifts. In my server, queue state is the most sensitive because fairness is part of the promise. A player who waited thirty minutes should not lose that place because Redis restarted.</p>
  <p class="ko">웹에서도 비슷하다. 세션 캐시가 날아가면 다시 로그인하면 된다. 불편하지만 복구 가능하다. 반면 작업 큐가 날아가면 작업이 유실된다. 레이트 리미터 카운터가 날아가면 공정성이 깨진다. 내 서버에서는 특히 큐 상태가 민감하다. 30분 동안 줄 서 있던 플레이어가 Redis 재시작 때문에 방금 온 사람보다 뒤로 밀리면, 그건 단순 불편이 아니라 공정성 훼손이다.</p>
</div>

<div class="bilingual">
  <p class="en">So every Redis key belongs to one of two classes.</p>
  <p class="ko">그래서 모든 Redis 키를 두 가지로 분류한다.</p>
</div>

<div class="table-wrap bilingual">
  <div class="en">
    <table>
      <thead>
        <tr>
          <th>Data</th>
          <th>Role</th>
          <th>If lost</th>
          <th>Eviction policy</th>
        </tr>
      </thead>
      <tbody>
        <tr>
          <td><code>{control}:queue_gate_state</code></td>
          <td><strong>Authority</strong></td>
          <td>Queue fairness breaks</td>
          <td><code>noeviction</code></td>
        </tr>
        <tr>
          <td><code>{queue:{nid}}:enter</code></td>
          <td><strong>Authority</strong></td>
          <td>Ordering resets</td>
          <td><code>noeviction</code></td>
        </tr>
        <tr>
          <td><code>{party:{pid}}</code></td>
          <td><strong>Authority</strong></td>
          <td>Party dissolves</td>
          <td><code>noeviction</code></td>
        </tr>
        <tr>
          <td><code>{sess:{sid}}:active</code></td>
          <td>Cache</td>
          <td>Recover from PostgreSQL</td>
          <td>TTL</td>
        </tr>
        <tr>
          <td><code>{char:{cid}}:name_cache</code></td>
          <td>Cache</td>
          <td>Recover from PostgreSQL</td>
          <td>LRU</td>
        </tr>
      </tbody>
    </table>
  </div>
  <div class="ko">
    <table>
      <thead>
        <tr>
          <th>데이터</th>
          <th>역할</th>
          <th>유실 시</th>
          <th>eviction 정책</th>
        </tr>
      </thead>
      <tbody>
        <tr>
          <td><code>{control}:queue_gate_state</code></td>
          <td><strong>Authority</strong></td>
          <td>대기열 공정성 파괴</td>
          <td><code>noeviction</code></td>
        </tr>
        <tr>
          <td><code>{queue:{nid}}:enter</code></td>
          <td><strong>Authority</strong></td>
          <td>순서 초기화</td>
          <td><code>noeviction</code></td>
        </tr>
        <tr>
          <td><code>{party:{pid}}</code></td>
          <td><strong>Authority</strong></td>
          <td>파티 해산</td>
          <td><code>noeviction</code></td>
        </tr>
        <tr>
          <td><code>{sess:{sid}}:active</code></td>
          <td>Cache</td>
          <td>PostgreSQL에서 재조회</td>
          <td>TTL</td>
        </tr>
        <tr>
          <td><code>{char:{cid}}:name_cache</code></td>
          <td>Cache</td>
          <td>PostgreSQL에서 재조회</td>
          <td>LRU</td>
        </tr>
      </tbody>
    </table>
  </div>
</div>

<div class="bilingual">
  <p class="en"><strong>Authority data</strong> means Redis is the only source of truth for that state. If the data vanishes, the state cannot be faithfully reconstructed or fairness is broken even if it can be approximated. <strong>Cache data</strong> means PostgreSQL remains the source of truth and Redis is only the fast path.</p>
  <p class="ko"><strong>Authority 데이터</strong>는 Redis가 그 상태의 유일한 진실이라는 뜻이다. 날아가면 충실한 복구가 불가능하거나, 대충 복구해도 공정성 같은 성질이 깨진다. <strong>Cache 데이터</strong>는 PostgreSQL이 원본이고 Redis는 빠른 경로일 뿐이다.</p>
</div>

<div class="bilingual">
  <h3 class="en">Why party state lives in Redis authority</h3>
  <h3 class="ko">왜 파티가 Redis Authority인가</h3>
</div>

<div class="bilingual">
  <p class="en">Party operations are highly concurrent: join, leave, leader transfer, dissolve. Those can all arrive in overlapping windows from multiple clients. PostgreSQL can handle it, but row-lock contention starts to matter. Redis Lua scripts solve the immediate atomicity problem elegantly.</p>
  <p class="ko">파티 변경은 동시에 많이 일어난다. 가입, 탈퇴, 리더 위임, 해산이 여러 클라이언트에서 겹친다. PostgreSQL로도 처리할 수 있지만, 이 경우 Row Lock 경합이 빠르게 커진다. Redis Lua 스크립트는 원자성 문제를 꽤 깔끔하게 해결해준다.</p>
</div>

```lua
local current = redis.call('HGET', KEYS[1], 'member_count')
if tonumber(current) >= tonumber(ARGV[2]) then
    return {0, 'ERR_FULL'}
end
redis.call('ZADD', KEYS[2], ARGV[3], ARGV[1])
redis.call('HINCRBY', KEYS[1], 'member_count', 1)
redis.call('INCR', KEYS[3])
return {1, 'OK'}
```

<div class="bilingual">
  <p class="en">This is the same pattern web systems use for atomic inventory decrement in Redis Lua. The difference is that game-party state is not just a backend record. It is a live multiplayer fact multiple humans observe at the same time. If party state becomes inconsistent, the failure is visible immediately in everyone's client.</p>
  <p class="ko">이건 웹 서비스에서 재고 차감을 Redis Lua로 원자적으로 처리하는 것과 같은 패턴이다. 다만 게임에서 파티 상태는 단순한 백엔드 레코드가 아니라 여러 사람이 동시에 보고 있는 실시간 멀티플레이 상태다. 파티 상태가 틀어지면 그 불일치는 즉시 각 플레이어의 화면에서 체감된다.</p>
</div>

<div class="bilingual">
  <p class="en">The cost is serialization. Redis is single-threaded, and Lua blocks other commands while running. In one four-node cluster test, party RPC p99 climbed to 1,332 ms and the slow log showed peaks above 7,402 ms. That tradeoff is not fully solved yet. If I had a larger team, I would consider moving party state to its own service. As a solo developer, I am still accepting Redis Lua's serial bottleneck because one more service also has a cost.</p>
  <p class="ko">대가는 직렬화다. Redis는 단일 스레드이고, Lua 스크립트는 실행 중 다른 명령을 블로킹한다. 4노드 클러스터 테스트에서는 파티 RPC p99가 1,332ms까지 튄 적이 있고, 느린 로그에는 7,402ms가 찍혔다. 이건 아직 완전히 해결하지 못한 트레이드오프다. 팀이 있었다면 파티 상태를 전용 서비스로 빼는 선택도 검토했을 것이다. 하지만 혼자 운영하는 입장에서는 또 하나의 서비스를 띄우는 비용도 만만치 않아서, 아직은 Redis Lua의 직렬 병목을 감수하고 있다.</p>
</div>

<div class="bilingual">
  <h3 class="en">Cluster-readiness: hash tags</h3>
  <h3 class="ko">클러스터 대비: Hash Tag</h3>
</div>

<div class="bilingual">
  <p class="en">The curly braces in keys like <code>{party:{pid}}</code> are there for Redis Cluster hash tags. Lua scripts touching multiple keys must operate on keys that live in the same slot. With hash tags, all party-related keys land together.</p>
  <p class="ko"><code>{party:{pid}}</code>처럼 중괄호를 쓰는 이유는 Redis Cluster의 hash tag 때문이다. Lua 스크립트가 여러 키를 건드릴 때는 그 키들이 같은 slot에 있어야 한다. hash tag를 사용하면 파티 관련 키를 같은 slot에 모을 수 있다.</p>
</div>

```go
PartyBaseKey(pid)     // {party:{pid}}
PartyMembersKey(pid)  // {party:{pid}}:members
PartyVersionKey(pid)  // {party:{pid}}:version
```

<div class="bilingual">
  <p class="en">I currently run a single Redis instance, but this key design means I do not have to redesign the naming scheme later just to become cluster-compatible.</p>
  <p class="ko">지금은 단일 Redis 인스턴스를 쓰고 있지만, 이 키 구조 덕분에 나중에 클러스터로 갈 때 이름 체계를 다시 설계할 필요가 없다.</p>
</div>

<div class="bilingual">
  <h2 class="en">WorldRPC: a four-lane classification system</h2>
  <h2 class="ko">WorldRPC: 4차선 분류 시스템</h2>
</div>

<div class="bilingual">
  <p class="en">Once GAME is forbidden from writing directly to the database, all state change flows through WORLD RPCs. The problem is that “send an RPC” is not one kind of action. A read, a payment-like commit, a bootstrap request, and a background save all deserve different retry and blocking rules.</p>
  <p class="ko">GAME에서 DB에 직접 쓰지 않으니, 모든 상태 변경은 WORLD RPC를 거쳐야 한다. 문제는 “RPC를 보낸다”는 행위가 다 같은 성질이 아니라는 점이다. 조회, 결제 같은 커밋, 부트스트랩, 백그라운드 저장은 모두 다른 재시도/대기 정책이 필요하다.</p>
</div>

<div class="bilingual">
  <p class="en">The web analogy is HTTP method safety plus one extra dimension. <code>GET</code> is safe to retry. <code>POST /payment</code> is not, unless you add idempotency. In games, there is also the question of whether the match loop can afford to wait for the answer right now.</p>
  <p class="ko">웹 개발로 비유하면 HTTP 메서드의 안전성(safe)/멱등성(idempotent) 구분과 비슷하다. <code>GET</code>은 자유롭게 재시도할 수 있지만, <code>POST /payment</code>는 멱등성 키 없이 재시도하면 이중 결제가 된다. 게임에서는 여기에 “지금 매치 루프가 이 응답을 기다릴 수 있는가?”라는 차원이 하나 더 붙는다.</p>
</div>

<div class="bilingual">
  <p class="en">So I split GAME to WORLD calls into four lanes.</p>
  <p class="ko">그래서 GAME → WORLD RPC를 네 가지 차선으로 분류했다.</p>
</div>

<div class="table-wrap bilingual">
  <div class="en">
    <table>
      <thead>
        <tr>
          <th>Lane</th>
          <th>Meaning</th>
          <th>Retry</th>
          <th>Idempotency</th>
          <th>Web analogy</th>
          <th>Example</th>
        </tr>
      </thead>
      <tbody>
        <tr>
          <td><strong>Query</strong></td>
          <td>Read-only</td>
          <td>Free</td>
          <td>Implicit</td>
          <td>GET</td>
          <td><code>world_inventory_list</code></td>
        </tr>
        <tr>
          <td><strong>Sync</strong></td>
          <td>Needs immediate result</td>
          <td>Limited</td>
          <td>Safe / CAS-style</td>
          <td>POST /login</td>
          <td><code>world_player_bootstrap</code></td>
        </tr>
        <tr>
          <td><strong>Commit</strong></td>
          <td>Economy mutation, exactly-once intent</td>
          <td>Only with same key</td>
          <td>Strong</td>
          <td>POST /payment</td>
          <td><code>world_trade_commit</code></td>
        </tr>
        <tr>
          <td><strong>Async</strong></td>
          <td>Fire and forget</td>
          <td>At-least-once</td>
          <td>Latest-wins</td>
          <td>Async queue publish</td>
          <td><code>world_save_position</code></td>
        </tr>
      </tbody>
    </table>
  </div>
  <div class="ko">
    <table>
      <thead>
        <tr>
          <th>차선</th>
          <th>의미</th>
          <th>재시도</th>
          <th>멱등성</th>
          <th>웹 비유</th>
          <th>예시</th>
        </tr>
      </thead>
      <tbody>
        <tr>
          <td><strong>Query</strong></td>
          <td>읽기 전용</td>
          <td>자유</td>
          <td>암묵적</td>
          <td>GET 요청</td>
          <td><code>world_inventory_list</code></td>
        </tr>
        <tr>
          <td><strong>Sync</strong></td>
          <td>즉시 결과 필요</td>
          <td>제한적</td>
          <td>안전 / CAS류</td>
          <td>POST /login</td>
          <td><code>world_player_bootstrap</code></td>
        </tr>
        <tr>
          <td><strong>Commit</strong></td>
          <td>경제 변동, exactly-once 의도</td>
          <td>같은 키로만</td>
          <td>강함</td>
          <td>POST /payment</td>
          <td><code>world_trade_commit</code></td>
        </tr>
        <tr>
          <td><strong>Async</strong></td>
          <td>fire-and-forget</td>
          <td>at-least-once</td>
          <td>latest-wins</td>
          <td>비동기 큐 발행</td>
          <td><code>world_save_position</code></td>
        </tr>
      </tbody>
    </table>
  </div>
</div>

<div class="bilingual">
  <p class="en"><strong>Query</strong> is for pure reads. If the call fails, it can simply be retried. <strong>Sync</strong> is for operations like character bootstrap that need an answer immediately because the player cannot proceed without it. <strong>Commit</strong> is for economy mutation: gold grants, trade completion, drops. These require strong idempotency because executing them twice is unacceptable. <strong>Async</strong> is for operations like position save where failure should not freeze the match loop.</p>
  <p class="ko"><strong>Query</strong>는 읽기 전용이다. 실패하면 그냥 다시 보내면 된다. <strong>Sync</strong>는 캐릭터 부트스트랩처럼 즉시 결과가 필요한 연산이다. <strong>Commit</strong>은 골드 지급, 거래 완료, 아이템 드롭처럼 경제가 걸린 연산이다. 두 번 실행되면 안 되므로 강한 멱등성이 필요하다. <strong>Async</strong>는 위치 저장처럼 실패해도 게임 전체를 멈추면 안 되는 연산이다.</p>
</div>

<div class="bilingual">
  <p class="en">Without this split, I would be forced into one of two bad extremes: treat everything like a payment and block on everything, or treat everything like telemetry and risk duplicate economy changes. The four lanes are my way of choosing a fitting safety level for each operation instead of pretending all calls are alike.</p>
  <p class="ko">이 분류가 없으면 두 가지 극단 중 하나를 택해야 한다. 모든 RPC를 결제처럼 취급해 전부 동기로 기다리거나, 반대로 전부 Async로 던져서 경제 연산까지 중복 실행 위험을 감수하는 것이다. 4차선 분류는 연산마다 적절한 안전 수준을 고르는 장치다.</p>
</div>

<div class="bilingual">
  <h3 class="en">Circuit breakers per RPC</h3>
  <h3 class="ko">회로 차단기(Circuit Breaker)</h3>
</div>

<div class="bilingual">
  <p class="en">Each RPC also has its own circuit breaker. The model is familiar from Hystrix or Resilience4j.</p>
  <p class="ko">각 RPC에는 독립적인 회로 차단기가 있다. Hystrix나 Resilience4j와 비슷한 모델이다.</p>
</div>

```text
Closed -> consecutive failures -> Open -> cooldown -> Half-Open -> success -> Closed
```

<div class="bilingual">
  <p class="en">The important part is that the breakers are <strong>independent</strong>. If <code>world_trade_commit</code> starts failing, <code>world_inventory_list</code> should still remain healthy. Without that separation, one slow or broken path can drag the whole game into a cascading failure. In web services, circuit breakers are often about latency and resilience. In a game server, they are also about keeping the match loop alive. A three-second API delay in a web page is a spinner. A three-second stall inside a match loop is visible lag for everyone in the map.</p>
  <p class="ko">핵심은 회로 차단기가 <strong>독립적</strong>이라는 점이다. <code>world_trade_commit</code>이 느려져도 <code>world_inventory_list</code>는 정상 동작해야 한다. 분리가 없으면 하나의 병목이 전체 게임을 먹통으로 만드는 연쇄 장애가 생긴다. 웹 서비스에서 Circuit Breaker가 보통 응답 시간과 복원력을 위한 것이라면, 게임 서버에서는 매치 루프의 생존을 위한 장치이기도 하다. 웹에서 3초 지연은 로딩 스피너지만, 매치 루프에서 3초 정지는 모든 플레이어가 동시에 체감하는 렉이다.</p>
</div>

<div class="bilingual">
  <h3 class="en">Deferred RPC: how not to block the match loop</h3>
  <h3 class="ko">Deferred RPC: 매치 루프를 블로킹하지 않는 법</h3>
</div>

<div class="bilingual">
  <p class="en">One of the main benefits of the lane system is that <strong>Async lane</strong> calls do not block the match loop.</p>
  <p class="ko">4차선 분류의 핵심 효과 중 하나는 <strong>Async 차선</strong>의 RPC가 매치 루프를 블로킹하지 않는다는 점이다.</p>
</div>

```text
One tick of the match loop (50 ms budget):
  Phase 1: handle player messages
  Phase 2: run NPC / monster ticks
  Phase 3: send Commit RPCs and wait if needed
  Phase 4: drain Async results from previous ticks
  Phase 5: flush outgoing broadcasts
```

<div class="bilingual">
  <p class="en">Commit calls have to wait because gold grants and trade commits must know whether they succeeded. Position save does not deserve the same privilege. That difference is the same as a synchronous API call versus a queue publish in a web architecture.</p>
  <p class="ko">Commit은 기다려야 한다. 골드 지급과 거래 완료는 성공 여부를 알아야 하기 때문이다. 하지만 위치 저장은 그렇게까지 할 필요가 없다. 이 차이는 웹에서 동기 API 호출과 메시지 큐 발행의 차이와 비슷하다.</p>
</div>

<div class="bilingual">
  <h2 class="en">Single Writer: lock-free concurrency</h2>
  <h2 class="ko">Single Writer: 락 없는 동시성</h2>
</div>

<div class="bilingual">
  <p class="en">Nakama matches are single-goroutine by default. That is a constraint, but also a powerful guarantee: if only one goroutine writes match state, I do not need mutexes around that state.</p>
  <p class="ko">Nakama 매치는 기본적으로 싱글 고루틴이다. 이건 제약이지만 동시에 강한 보장이기도 하다. 매치 상태를 오직 하나의 고루틴만 쓴다면, 그 상태를 보호하는 뮤텍스가 필요 없다.</p>
</div>

<div class="bilingual">
  <h3 class="en">The problem</h3>
  <h3 class="ko">문제</h3>
</div>

<div class="bilingual">
  <p class="en">Reality is still messier than that. If I simulate ten NPC AIs serially, tick time grows with NPC count. If I send ten WorldRPC calls serially, even 5 ms per call burns 50 ms, which can consume the entire tick budget. Parallelism is necessary.</p>
  <p class="ko">하지만 현실은 그렇게 단순하지 않다. NPC 10마리의 AI를 순차적으로 돌리면 NPC 수에 비례해서 틱 시간이 늘어난다. WorldRPC 10개를 순차 호출하면 5ms씩만 걸려도 50ms가 날아간다. 병렬 처리가 필요하다.</p>
</div>

<div class="bilingual">
  <p class="en">I felt this sharply during a 32K CCU test. <code>MatchJoin</code> took about 20 ms per player. Five players joining at once turned into roughly 100 ms, which meant the tick budget was already gone before the rest of the simulation even ran. Then pong timeouts fired, reconnect storms started, and the reconnects themselves created more <code>MatchJoin</code> work. The whole failure chain reinforced an old habit of mine: if a small fallback can collapse a live service, I want to break that scenario down until I understand every link in the chain.</p>
  <p class="ko">이걸 뼈저리게 느낀 순간이 32K CCU 테스트였다. <code>MatchJoin</code> 처리에 플레이어당 20ms 정도가 걸렸는데, 5명이 동시에 입장하면 100ms가 된다. 틱 예산을 입장 처리만으로 다 써버리는 셈이다. 그러자 pong 타임아웃이 터지고, 재접속 폭주가 생기고, 재접속이 다시 <code>MatchJoin</code>을 만들어내는 연쇄 붕괴가 시작됐다. 작은 fallback 하나가 라이브 서비스 전체를 무너뜨릴 수 있다면, 그 사고 흐름을 끝까지 잘게 쪼개서 이해하고 싶어지는 습관이 있다.</p>
</div>

<div class="bilingual">
  <p class="en">I wrote about one version of that chaos in <a class="inline-link" href="{{ '/server-engineering/2026/04/03/ai-could-not-find-the-bug/' | relative_url }}">a previous post</a>. Queue admission solved that particular issue, but it reinforced the larger lesson: tiny runtime assumptions become system-wide failures under load.</p>
  <p class="ko">이런 카오스 시나리오의 한 버전은 <a class="inline-link" href="{{ '/server-engineering/2026/04/03/ai-could-not-find-the-bug/' | relative_url }}">이전 글</a>에서도 다뤘다. Queue admission이 그 문제 자체는 해결했지만, 작은 런타임 가정이 부하 상황에서는 전체 장애로 번진다는 감각은 더 강해졌다.</p>
</div>

<div class="bilingual">
  <p class="en">In Go, the obvious answer is goroutines plus mutexes. The problem is that Go maps are not safe for concurrent write. Even when separate goroutines touch different logical keys, internal resizing can still race. Fine-grained locking sounds nice until it creates deadlocks or erases the benefit of parallel work.</p>
  <p class="ko">Go에서 병렬 처리는 보통 고루틴과 뮤텍스로 푼다. 하지만 Go의 map은 concurrent write에 안전하지 않다. 서로 다른 키를 건드리더라도 내부 리사이징 타이밍에 레이스가 발생할 수 있다. 세밀한 락은 데드락/라이브락 위험을 만들거나, 병렬 처리의 의미를 지워버리기 쉽다.</p>
</div>

<div class="bilingual">
  <h3 class="en">The decision</h3>
  <h3 class="ko">결정</h3>
</div>

<div class="bilingual">
  <p class="en">I use a <strong>Single Writer</strong> pattern with three phases.</p>
  <p class="ko"><strong>Single Writer</strong> 패턴을 쓴다. 3단계로 나눈다.</p>
</div>

```text
Phase 1: Prepare (serial, main goroutine)
  validate work
  read from matchState
  build read-only snapshots

Phase 2: Parallel Execute (worker goroutines)
  run WorldRPC calls, NPC AI, calculations
  read snapshots only
  never mutate matchState
  collect results into channels or slices

Phase 3: Apply (serial, main goroutine)
  merge results back into matchState
```

<div class="bilingual">
  <p class="en">The closest web analogy is React state updates or event sourcing. Workers collect intended changes, then one authority applies them in order. That lets me keep match state mutation single-threaded while still parallelizing expensive external work.</p>
  <p class="ko">웹 개발의 비유로는 React의 상태 업데이트 모델이나 이벤트 소싱에 가깝다. 변경 의도를 먼저 수집하고, 마지막에 하나의 권위가 순서대로 상태에 반영한다. 덕분에 비싼 외부 작업은 병렬화하면서도 매치 상태 변경 자체는 단일 스레드로 유지할 수 있다.</p>
</div>

<div class="bilingual">
  <p class="en">A simplified inventory-move example looks like this.</p>
  <p class="ko">인벤토리 이동 10건을 병렬 처리하는 단순화된 예시는 이렇다.</p>
</div>

```go
tasks := validateMoves(matchState, requests)

results := make([]MoveResult, len(tasks))
var wg sync.WaitGroup
sem := make(chan struct{}, 16)

for i, task := range tasks {
    wg.Add(1)
    sem <- struct{}{}
    go func(idx int, t MoveTask) {
        defer wg.Done()
        defer func() { <-sem }()
        results[idx] = callWorldRPC(t)
    }(i, task)
}

wg.Wait()

for i, result := range results {
    applyToMatchState(matchState, result)
}
```

<div class="bilingual">
  <p class="en">The absolute rule is this: <strong>only the main goroutine writes to <code>matchState</code></strong>. Once that rule is true, mutexes around the main match state mostly disappear.</p>
  <p class="ko">절대 규칙은 하나다. <strong><code>matchState</code>에 쓰는 건 오직 메인 고루틴</strong>. 이 규칙만 지키면 메인 매치 상태를 보호하는 뮤텍스는 대부분 필요 없어진다.</p>
</div>

<div class="bilingual">
  <p class="en">Could I just use mutexes instead? In theory, yes. In practice, the state is built around Go maps, and Go maps do not tolerate concurrent writes safely. So the choice becomes either coarse locks that erase parallelism or a structure that avoids shared writes in the first place. Single Writer is the “avoid shared writes” answer.</p>
  <p class="ko">그냥 뮤텍스를 쓰면 안 되냐고 물을 수 있다. 이론적으로는 가능하다. 하지만 상태는 Go map을 중심으로 되어 있고, Go map은 concurrent write에 안전하지 않다. 결국 선택지는 병렬성을 지워버리는 거친 락이거나, 아예 공유 쓰기를 피하는 구조다. Single Writer는 “락을 잘 거는 것”보다 “락이 필요 없는 구조를 만든다”에 가깝다.</p>
</div>

<div class="bilingual">
  <h3 class="en">Partitioned write: a special optimization</h3>
  <h3 class="ko">파티션 쓰기: 특수 최적화</h3>
</div>

<div class="bilingual">
  <p class="en">Monster or NPC AI can sometimes go further. If each goroutine is guaranteed to mutate only its own entity and never shared state, then the serial apply phase can be skipped for that subsystem. That is a narrower, more dangerous optimization, but when the partition guarantee really holds, it is worth it.</p>
  <p class="ko">몬스터나 NPC AI는 경우에 따라 더 나아갈 수 있다. 각 고루틴이 자기 엔티티의 필드만 바꾸고 공유 상태를 건드리지 않는다는 보장이 있으면, 그 부분은 직렬 Apply 단계를 건너뛸 수 있다. 범위가 더 좁고 더 위험한 최적화지만, 파티션 보장이 진짜로 성립한다면 가치가 있다.</p>
</div>

```go
func TickMonstersFullParallel(monsters []*Monster) {
    for _, m := range monsters {
        go m.Tick()
    }
}
```

<div class="bilingual">
  <p class="en">Without that kind of optimization, one hundred monsters at one millisecond each already turn into a 100 ms serial budget blowout. With partition-safe parallelism, the cost can scale closer to available cores.</p>
  <p class="ko">이런 최적화가 없으면 몬스터 100마리를 1ms씩만 돌려도 순차 처리만으로 100ms가 된다. 파티션이 안전하게 보장된다면 코어 수에 맞춰 훨씬 줄일 수 있다.</p>
</div>

<div class="bilingual">
  <h2 class="en">Summary of part 1</h2>
  <h2 class="ko">정리: 1편의 핵심 결정들</h2>
</div>

<div class="table-wrap bilingual">
  <div class="en">
    <table>
      <thead>
        <tr>
          <th>Decision</th>
          <th>Main motivation</th>
          <th>Cost</th>
        </tr>
      </thead>
      <tbody>
        <tr>
          <td>Nakama + Go plugin</td>
          <td>Save major infrastructure effort, keep native performance</td>
          <td>Single-goroutine match constraint</td>
        </tr>
        <tr>
          <td><strong>Map = match hack</strong></td>
          <td>Avoid writing a world server from scratch</td>
          <td>Had to build GC, singleton protection, and routing myself</td>
        </tr>
        <tr>
          <td>GAME / WORLD / CONTROL split</td>
          <td>Centralize consistency points and domain boundaries</td>
          <td>RPC latency and call complexity</td>
        </tr>
        <tr>
          <td>Citus sharding by <code>user_id</code></td>
          <td>Horizontal scale for inventory while keeping PostgreSQL</td>
          <td>2PC cost for cross-shard trade</td>
        </tr>
        <tr>
          <td>Redis authority/cache split</td>
          <td>Make loss impact explicit and recovery policy intentional</td>
          <td>More complex key design and policy management</td>
        </tr>
        <tr>
          <td>WorldRPC 4 lanes</td>
          <td>Choose fitting safety level per operation, protect the loop</td>
          <td>More concepts to learn and maintain</td>
        </tr>
        <tr>
          <td>Single Writer</td>
          <td>Lock-free parallelism and simpler state safety</td>
          <td>Three-phase structure and more orchestration code</td>
        </tr>
      </tbody>
    </table>
  </div>
  <div class="ko">
    <table>
      <thead>
        <tr>
          <th>결정</th>
          <th>핵심 동기</th>
          <th>대가</th>
        </tr>
      </thead>
      <tbody>
        <tr>
          <td>Nakama + Go 플러그인</td>
          <td>인프라 80% 절약, 네이티브 성능 확보</td>
          <td>싱글 고루틴 매치 제약</td>
        </tr>
        <tr>
          <td><strong>맵 = 매치 꼼수</strong></td>
          <td>월드 서버를 바닥부터 안 짜도 됨</td>
          <td>GC, 싱글톤 보장, 라우팅을 직접 구현해야 함</td>
        </tr>
        <tr>
          <td>3계층 분리 (GAME / WORLD / CONTROL)</td>
          <td>일관성 보장점 집중, 도메인 경계 명확화</td>
          <td>RPC 지연, 호출 복잡도</td>
        </tr>
        <tr>
          <td>Citus 샤딩 (<code>user_id</code>)</td>
          <td>인벤토리 수평 확장, PostgreSQL 생태계 유지</td>
          <td>크로스 샤드 거래 2PC 비용</td>
        </tr>
        <tr>
          <td>Redis Authority / Cache 구분</td>
          <td>유실 시 영향과 복구 전략을 명확히 함</td>
          <td>키 설계와 정책 관리 복잡도 증가</td>
        </tr>
        <tr>
          <td>WorldRPC 4차선</td>
          <td>연산별 최적 안전 수준 선택, 매치 루프 보호</td>
          <td>분류 기준 학습 비용, 유지 비용</td>
        </tr>
        <tr>
          <td>Single Writer</td>
          <td>락 없는 병렬성, 상태 안전성 단순화</td>
          <td>3단계 구조 강제, 오케스트레이션 코드 증가</td>
        </tr>
      </tbody>
    </table>
  </div>
</div>

<div class="bilingual">
  <p class="en">These decisions were not independent. “Map equals match” created the performance requirements of the match loop. That led to the GAME / WORLD / CONTROL split. That split led to WorldRPC. WorldRPC led to the four-lane system. One decision kept creating the next one.</p>
  <p class="ko">이 결정들은 서로 독립적이지 않았다. “맵 = 매치” 꼼수가 매치 루프의 성능 요구를 만들었고, 그게 GAME / WORLD / CONTROL 분리를 낳았고, 그 분리가 WorldRPC를 만들었고, WorldRPC가 4차선 분류를 만들었다. 하나의 결정이 다음 결정을 계속 만들어냈다.</p>
</div>

<div class="bilingual">
  <p class="en">Part 2 covers how the server survives load on top of this base: queue admission, the Lua DSL layer, the OpsEngine, and the cluster strategy.</p>
  <p class="ko">2편에서는 이 기반 위에서 부하를 어떻게 버티는지, 즉 Queue Admission, Lua DSL, OpsEngine, 클러스터 전략을 다룬다.</p>
</div>

<div class="bilingual">
  <p class="en"><em>Continued in Part 2.</em></p>
  <p class="ko"><em>2편에서 계속.</em></p>
</div>
