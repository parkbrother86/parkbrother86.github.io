---
title: Home | 홈
---

<section class="hero-grid">
  <article class="hero hero-primary">
    <span class="eyebrow">Bilingual Journal</span>
    <h1 class="hero-title bilingual">
      <span class="en">Thoughts, sketches, and personal notes with a sharper editorial feel.</span>
      <span class="ko">생각과 기록, 개인적인 메모를 조금 더 잡지 같은 분위기로 담아내는 공간.</span>
    </h1>
    <p class="hero-copy bilingual">
      <span class="en">This site is designed to feel like an online notebook rather than a plain template: quiet typography, richer spacing, and a layout that gives every post a little more presence.</span>
      <span class="ko">이 사이트는 단순한 템플릿보다 온라인 노트에 가깝게 보이도록 구성했습니다. 차분한 타이포그래피와 여백, 그리고 글 하나하나가 더 도드라지도록 하는 레이아웃을 담았습니다.</span>
    </p>
    <div class="hero-actions">
      <a class="button primary" href="{{ '/about/' | relative_url }}">
        <span class="bilingual">
          <span class="en">Read the Introduction</span>
          <span class="ko">소개 읽기</span>
        </span>
      </a>
      <a class="button secondary" href="https://github.com/parkbrother86/parkbrother86.github.io">
        <span class="bilingual">
          <span class="en">Open Repository</span>
          <span class="ko">저장소 열기</span>
        </span>
      </a>
    </div>
  </article>

  <aside class="hero hero-side">
    <div class="metric-card">
      <span class="metric-label bilingual">
        <span class="en">Current mood</span>
        <span class="ko">현재 분위기</span>
      </span>
      <strong>Editorial / Reflective</strong>
    </div>
    <div class="metric-card">
      <span class="metric-label bilingual">
        <span class="en">Language</span>
        <span class="ko">언어</span>
      </span>
      <strong>English + Korean</strong>
    </div>
    <div class="metric-note bilingual">
      <span class="en">A flexible home for posts, project notes, reading logs, and small updates.</span>
      <span class="ko">포스트, 프로젝트 기록, 읽은 것들, 짧은 업데이트를 함께 담아갈 수 있는 유연한 홈입니다.</span>
    </div>
  </aside>
</section>

<section class="feature-grid">
  <article class="panel feature-panel">
    <span class="feature-number">01</span>
    <h2 class="section-title bilingual">
      <span class="en">What lives here</span>
      <span class="ko">여기에 담을 것</span>
    </h2>
    <p class="bilingual">
      <span class="en">Personal essays, build notes, project snapshots, and the kind of short thoughts that usually disappear into drafts.</span>
      <span class="ko">개인 에세이, 작업 메모, 프로젝트 스냅샷, 그리고 보통 초안 속에 머무는 짧은 생각들을 이곳에 모아둘 수 있습니다.</span>
    </p>
  </article>
  <article class="panel feature-panel accent-panel">
    <span class="feature-number">02</span>
    <h2 class="section-title bilingual">
      <span class="en">How it should feel</span>
      <span class="ko">어떤 느낌으로 보일지</span>
    </h2>
    <p class="bilingual">
      <span class="en">Less like a starter theme, more like a carefully kept desk: warm tones, generous rhythm, and room for words to breathe.</span>
      <span class="ko">기본 테마보다는 잘 정돈된 책상처럼 보이도록, 따뜻한 톤과 넉넉한 여백, 숨 쉴 수 있는 문장 배치를 목표로 했습니다.</span>
    </p>
  </article>
</section>

<section class="section-head">
  <div>
    <span class="eyebrow">Latest Writing</span>
    <h2 class="section-title bilingual">
      <span class="en">Recent Posts</span>
      <span class="ko">최근 글</span>
    </h2>
  </div>
</section>

{% if site.posts and site.posts.size > 0 %}
  <section class="post-list editorial-list">
    {% for post in site.posts limit: 5 %}
      <article class="post-card">
        <span class="post-date">{{ post.date | date: "%Y-%m-%d" }}</span>
        <h3 class="post-card-title"><a class="post-link" href="{{ post.url | relative_url }}">{{ post.title }}</a></h3>
        <p>{{ post.excerpt | strip_html | truncate: 150 }}</p>
        <a class="read-more" href="{{ post.url | relative_url }}">
          <span class="bilingual inline">
            <span class="en">Read entry</span>
            <span class="ko">글 읽기</span>
          </span>
        </a>
      </article>
    {% endfor %}
  </section>
{% else %}
  <section class="panel">
    <p class="empty-state bilingual">
      <span class="en">No posts yet. Add a Markdown file inside `_posts` to publish your first entry.</span>
      <span class="ko">아직 글이 없습니다. `_posts` 폴더에 Markdown 파일을 추가하면 첫 글을 올릴 수 있습니다.</span>
    </p>
  </section>
{% endif %}

<section class="panel closing-note">
  <p class="bilingual">
    <span class="en">If you want, this can keep evolving into something more specific next: a devlog, a reading journal, a portfolio-blog hybrid, or a cleaner minimal essay site.</span>
    <span class="ko">원하시면 다음 단계에서 이 공간을 개발 기록형, 독서 저널형, 포트폴리오 겸 블로그형, 혹은 더 미니멀한 에세이 사이트로 더 구체화할 수 있습니다.</span>
  </p>
</section>
