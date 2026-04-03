---
title: Home | 홈
---

<section class="hero">
  <span class="eyebrow">GitHub Pages</span>
  <h1 class="hero-title bilingual">
    <span class="en">Park Brother Blog</span>
    <span class="ko">박형제 블로그</span>
  </h1>
  <p class="bilingual">
    <span class="en">A Jekyll-powered starting point for a personal blog, ready to grow with new pages, posts, and custom sections.</span>
    <span class="ko">새 페이지와 포스트, 원하는 섹션을 계속 확장해 나갈 수 있는 Jekyll 기반 개인 블로그 시작점입니다.</span>
  </p>
  <div class="hero-actions">
    <a class="button primary" href="{{ '/about/' | relative_url }}">
      <span class="bilingual">
        <span class="en">About Page</span>
        <span class="ko">소개 페이지</span>
      </span>
    </a>
    <a class="button secondary" href="https://github.com/parkbrother86/parkbrother86.github.io">
      <span class="bilingual">
        <span class="en">GitHub Repository</span>
        <span class="ko">GitHub 저장소</span>
      </span>
    </a>
  </div>
</section>

<section class="panel">
  <h2 class="section-title bilingual">
    <span class="en">Recent Posts</span>
    <span class="ko">최근 글</span>
  </h2>
  {% if site.posts and site.posts.size > 0 %}
    <div class="post-list">
      {% for post in site.posts limit: 5 %}
        <article class="post-card">
          <span class="post-date">{{ post.date | date: "%Y-%m-%d" }}</span>
          <h3 class="post-card-title"><a class="post-link" href="{{ post.url | relative_url }}">{{ post.title }}</a></h3>
          <p>{{ post.excerpt | strip_html | truncate: 120 }}</p>
        </article>
      {% endfor %}
    </div>
  {% else %}
    <p class="empty-state bilingual">
      <span class="en">No posts yet. Add a Markdown file inside `_posts` to publish your first entry.</span>
      <span class="ko">아직 글이 없습니다. `_posts` 폴더에 Markdown 파일을 추가하면 첫 글을 올릴 수 있습니다.</span>
    </p>
  {% endif %}
</section>
