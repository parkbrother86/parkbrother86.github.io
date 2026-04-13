---
title: Home
---

{% assign total_posts = site.posts | size %}
{% assign latest_post = site.posts | first %}

<section class="home-page">
  <header class="home-hero">
    <div class="home-hero-copy">
      <p class="page-kicker bilingual">
        <span class="en">Engineering notebook</span>
        <span class="ko">엔지니어링 노트북</span>
      </p>
      <h1 class="home-title bilingual">
        <span class="en">Clean, direct writing about building systems and shipping software.</span>
        <span class="ko">시스템을 만들고 소프트웨어를 운영하며 얻은 생각을 직관적으로 정리한 기록입니다.</span>
      </h1>
      <p class="home-intro bilingual">
        <span class="en">This front page is designed to feel more like a simple starting point than a dense archive: pick a topic, open the latest writing, and move around without hunting for structure.</span>
        <span class="ko">첫 화면은 복잡한 아카이브보다 명확한 시작점처럼 보이도록 구성했습니다. 주제를 고르고, 최신 글을 열고, 흐름을 잃지 않고 이동할 수 있게 만들었습니다.</span>
      </p>

      <div class="home-actions">
        <a class="primary-link bilingual inline" href="#topics">
          <span class="en">Browse topics</span>
          <span class="ko">주제 보기</span>
        </a>
        <a class="secondary-link bilingual inline" href="#recent">
          <span class="en">Latest posts</span>
          <span class="ko">최신 글 보기</span>
        </a>
        <a class="secondary-link bilingual inline" href="{{ '/about/' | relative_url }}">
          <span class="en">About me</span>
          <span class="ko">소개 보기</span>
        </a>
      </div>
    </div>

    <aside class="home-hero-panel">
      <p class="panel-kicker bilingual">
        <span class="en">Quick overview</span>
        <span class="ko">한눈에 보기</span>
      </p>

      <div class="hero-stat-list">
        <article class="hero-stat">
          <span class="hero-stat-label bilingual inline">
            <span class="en">Topics</span>
            <span class="ko">주제</span>
          </span>
          <strong class="hero-stat-value">{{ site.data.categories | size }}</strong>
        </article>
        <article class="hero-stat">
          <span class="hero-stat-label bilingual inline">
            <span class="en">Posts</span>
            <span class="ko">글</span>
          </span>
          <strong class="hero-stat-value">{{ total_posts }}</strong>
        </article>
        {% if latest_post %}
          <article class="hero-stat">
            <span class="hero-stat-label bilingual inline">
              <span class="en">Last update</span>
              <span class="ko">최근 갱신</span>
            </span>
            <strong class="hero-stat-value">{{ latest_post.date | date: "%Y.%m.%d" }}</strong>
          </article>
        {% endif %}
      </div>

      {% if latest_post %}
        <a class="panel-highlight" href="{{ latest_post.url | relative_url }}">
          <span class="panel-highlight-kicker bilingual inline">
            <span class="en">Latest update</span>
            <span class="ko">최근 업데이트</span>
          </span>
          <strong class="panel-highlight-title bilingual">
            <span class="en">{{ latest_post.title }}</span>
            <span class="ko">{{ latest_post.title_kr | default: latest_post.title }}</span>
          </strong>
          <span class="panel-highlight-meta">{{ latest_post.date | date: "%Y.%m.%d" }}</span>
        </a>
      {% endif %}
    </aside>
  </header>

  <section class="content-section" id="topics">
    <div class="section-header">
      <div>
        <p class="section-kicker bilingual">
          <span class="en">Topics</span>
          <span class="ko">주제</span>
        </p>
        <h2 class="section-title bilingual">
          <span class="en">Start from the subject you care about</span>
          <span class="ko">관심 있는 주제부터 바로 시작할 수 있습니다</span>
        </h2>
      </div>
      <p class="section-note bilingual">
        <span class="en">Each card points to a topic archive and surfaces its most recent post.</span>
        <span class="ko">각 카드는 카테고리 페이지로 연결되고, 가장 최근 글도 함께 보여줍니다.</span>
      </p>
    </div>

    <div class="card-grid topic-grid">
      {% for category in site.data.categories %}
        {% assign category_posts = site.categories[category.slug] %}
        {% assign category_page = site.pages | where: "category_slug", category.slug | first %}
        {% assign latest_category_post = category_posts | first %}
        <article class="content-card topic-card">
          <div class="topic-card-top">
            <span class="topic-number">{{ forloop.index | prepend: '0' }}</span>
            <span class="topic-count bilingual inline">
              <span class="en">{{ category_posts | size }} posts</span>
              <span class="ko">{{ category_posts | size }}개 글</span>
            </span>
          </div>

          <h3 class="card-title bilingual">
            <span class="en">{{ category.en }}</span>
            <span class="ko">{{ category.kr }}</span>
          </h3>

          {% if category_page %}
            <p class="card-text bilingual">
              <span class="en">{{ category_page.category_intro_en }}</span>
              <span class="ko">{{ category_page.category_intro_kr }}</span>
            </p>
          {% endif %}

          {% if latest_category_post %}
            <div class="topic-latest">
              <span class="topic-latest-label bilingual inline">
                <span class="en">Latest</span>
                <span class="ko">최신 글</span>
              </span>
              <a class="card-link bilingual" href="{{ latest_category_post.url | relative_url }}">
                <span class="en">{{ latest_category_post.title }}</span>
                <span class="ko">{{ latest_category_post.title_kr | default: latest_category_post.title }}</span>
              </a>
              <span class="card-date">{{ latest_category_post.date | date: "%Y.%m.%d" }}</span>
            </div>
          {% else %}
            <p class="card-text bilingual">
              <span class="en">No published posts yet.</span>
              <span class="ko">아직 공개된 글이 없습니다.</span>
            </p>
          {% endif %}

          <a class="secondary-link topic-action bilingual inline" href="{{ category.path | relative_url }}">
            <span class="en">Open topic</span>
            <span class="ko">주제 열기</span>
          </a>
        </article>
      {% endfor %}
    </div>
  </section>

  <section class="content-section" id="recent">
    <div class="section-header">
      <div>
        <p class="section-kicker bilingual">
          <span class="en">Recent writing</span>
          <span class="ko">최신 글</span>
        </p>
        <h2 class="section-title bilingual">
          <span class="en">Newest posts across the whole site</span>
          <span class="ko">사이트 전체의 최신 글을 한 번에 볼 수 있습니다</span>
        </h2>
      </div>
      <p class="section-note bilingual">
        <span class="en">A compact feed for people who prefer browsing by freshness instead of topic.</span>
        <span class="ko">주제보다 최신순으로 훑고 싶은 사람을 위한 간단한 목록입니다.</span>
      </p>
    </div>

    <div class="card-grid entry-grid">
      {% for post in site.posts limit: 6 %}
        {% assign post_category_slug = post.categories | first %}
        {% assign post_category = site.data.categories | where: "slug", post_category_slug | first %}
        <article class="content-card entry-card">
          <div class="entry-card-meta">
            {% if post_category %}
              <a class="meta-chip bilingual inline" href="{{ post_category.path | relative_url }}">
                <span class="en">{{ post_category.en }}</span>
                <span class="ko">{{ post_category.kr }}</span>
              </a>
            {% endif %}
            <span class="card-date">{{ post.date | date: "%Y.%m.%d" }}</span>
          </div>

          <h3 class="card-title large">
            <a href="{{ post.url | relative_url }}">
              <span class="bilingual">
                <span class="en">{{ post.title }}</span>
                <span class="ko">{{ post.title_kr | default: post.title }}</span>
              </span>
            </a>
          </h3>

          <p class="card-text bilingual">
            <span class="en">{{ post.excerpt_en | default: post.excerpt | strip_html | strip_newlines }}</span>
            <span class="ko">{{ post.excerpt_kr | default: post.excerpt | strip_html | strip_newlines }}</span>
          </p>

          <a class="text-link bilingual inline" href="{{ post.url | relative_url }}">
            <span class="en">Read post</span>
            <span class="ko">글 읽기</span>
          </a>
        </article>
      {% endfor %}
    </div>
  </section>
</section>
