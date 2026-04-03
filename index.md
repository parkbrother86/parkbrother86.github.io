---
title: Home
---

<section class="hero">
  <span class="eyebrow">GitHub Pages</span>
  <h1>Park Brother Blog</h1>
  <p>Jekyll 기반으로 확장할 수 있는 개인 블로그 시작점입니다. 소개 페이지, 포스트 목록, 공통 레이아웃과 스타일을 기본으로 포함합니다.</p>
  <div class="hero-actions">
    <a class="button primary" href="{{ '/about/' | relative_url }}">소개 보기</a>
    <a class="button secondary" href="https://github.com/parkbrother86/parkbrother86.github.io">GitHub 저장소</a>
  </div>
</section>

<section class="panel">
  <h2 class="section-title">최근 글</h2>
  {% if site.posts and site.posts.size > 0 %}
    <div class="post-list">
      {% for post in site.posts limit: 5 %}
        <article class="post-card">
          <span class="post-date">{{ post.date | date: "%Y-%m-%d" }}</span>
          <h3><a class="post-link" href="{{ post.url | relative_url }}">{{ post.title }}</a></h3>
          <p>{{ post.excerpt | strip_html | truncate: 120 }}</p>
        </article>
      {% endfor %}
    </div>
  {% else %}
    <p class="empty-state">아직 등록된 글이 없습니다. `_posts` 폴더에 Markdown 파일을 추가해 보세요.</p>
  {% endif %}
</section>
