---
title: Home
---

{% assign total_posts = site.posts | size %}

<section class="doc-page doc-home">
  <header class="doc-page-head">
    <p class="page-kicker bilingual">
      <span class="en">Documentation Index</span>
      <span class="ko">문서 색인</span>
    </p>
    <h1 class="page-title bilingual">
      <span class="en">Engineering notes organized like a working notebook.</span>
      <span class="ko">작업 노트처럼 정리한 엔지니어링 기록.</span>
    </h1>
    <p class="page-intro bilingual">
      <span class="en">This site is structured more like a documentation index than a lifestyle blog: sections, archives, recent entries, and bilingual notes arranged for scanning before reading.</span>
      <span class="ko">이 사이트는 라이프스타일 블로그보다 문서 인덱스에 가깝게 구성했습니다. 읽기 전에 훑을 수 있도록 섹션, 아카이브, 최근 글, 이중 언어 노트를 정리해 둡니다.</span>
    </p>

    <div class="doc-stat-grid">
      <article class="doc-stat">
        <span class="doc-stat-label bilingual">
          <span class="en">Tracks</span>
          <span class="ko">트랙</span>
        </span>
        <strong class="doc-stat-value">{{ site.data.categories | size }}</strong>
      </article>
      <article class="doc-stat">
        <span class="doc-stat-label bilingual">
          <span class="en">Entries</span>
          <span class="ko">글 수</span>
        </span>
        <strong class="doc-stat-value">{{ total_posts }}</strong>
      </article>
      <article class="doc-stat">
        <span class="doc-stat-label bilingual">
          <span class="en">Languages</span>
          <span class="ko">언어</span>
        </span>
        <strong class="doc-stat-value">2</strong>
      </article>
    </div>
  </header>

  <section class="doc-section">
    <div class="section-header">
      <div>
        <p class="section-kicker bilingual">
          <span class="en">Index</span>
          <span class="ko">색인</span>
        </p>
        <h2 class="section-title bilingual">
          <span class="en">Category map</span>
          <span class="ko">카테고리 맵</span>
        </h2>
      </div>
      <p class="section-note bilingual">
        <span class="en">Each track points to its latest published entry.</span>
        <span class="ko">각 트랙은 가장 최근에 발행한 글로 연결됩니다.</span>
      </p>
    </div>

    <div class="doc-index-table">
      <div class="doc-index-head">
        <span class="bilingual inline">
          <span class="en">ID</span>
          <span class="ko">번호</span>
        </span>
        <span class="bilingual inline">
          <span class="en">Track</span>
          <span class="ko">트랙</span>
        </span>
        <span class="bilingual inline">
          <span class="en">Latest Entry</span>
          <span class="ko">최신 글</span>
        </span>
        <span class="bilingual inline">
          <span class="en">Status</span>
          <span class="ko">상태</span>
        </span>
      </div>

      {% for category in site.data.categories %}
        {% assign latest_post = site.categories[category.slug] | first %}
        <article class="doc-index-row">
          <div class="doc-index-col doc-index-no">{{ forloop.index | prepend: '0' }}</div>
          <div class="doc-index-col">
            <a class="doc-track-link" href="{{ category.path | relative_url }}">
              <span class="bilingual">
                <span class="en">{{ category.en }}</span>
                <span class="ko">{{ category.kr }}</span>
              </span>
            </a>
          </div>
          <div class="doc-index-col">
            {% if latest_post %}
              <a class="doc-entry-link" href="{{ latest_post.url | relative_url }}">
                <span class="bilingual">
                  <span class="en">{{ latest_post.title }}</span>
                  {% if latest_post.title_kr %}
                    <span class="ko">{{ latest_post.title_kr }}</span>
                  {% endif %}
                </span>
              </a>
              <p class="doc-entry-excerpt bilingual">
                <span class="en">{{ latest_post.excerpt_en | default: latest_post.excerpt | strip_html | strip_newlines }}</span>
                <span class="ko">{{ latest_post.excerpt_kr | default: latest_post.excerpt | strip_html | strip_newlines }}</span>
              </p>
            {% else %}
              <p class="doc-entry-excerpt bilingual">
                <span class="en">No published entries yet.</span>
                <span class="ko">아직 발행된 글이 없습니다.</span>
              </p>
            {% endif %}
          </div>
          <div class="doc-index-col doc-index-meta">
            {% if latest_post %}
              <span class="doc-index-date bilingual">
                <span class="en">{{ latest_post.date | date: "%Y-%m-%d" }}</span>
                <span class="ko">{{ latest_post.date | date: "%Y-%m-%d" }}</span>
              </span>
              <a class="inline-link bilingual inline" href="{{ category.path | relative_url }}">
                <span class="en">Open track</span>
                <span class="ko">트랙 열기</span>
              </a>
            {% else %}
              <span class="doc-index-date bilingual">
                <span class="en">Empty</span>
                <span class="ko">비어 있음</span>
              </span>
              <a class="inline-link bilingual inline" href="{{ category.path | relative_url }}">
                <span class="en">Open track</span>
                <span class="ko">트랙 열기</span>
              </a>
            {% endif %}
          </div>
        </article>
      {% endfor %}
    </div>
  </section>

  <section class="doc-section">
    <div class="section-header">
      <div>
        <p class="section-kicker bilingual">
          <span class="en">Recent</span>
          <span class="ko">최근 글</span>
        </p>
        <h2 class="section-title bilingual">
          <span class="en">Latest entries</span>
          <span class="ko">최신 엔트리</span>
        </h2>
      </div>
      <p class="section-note bilingual">
        <span class="en">A running list across every track.</span>
        <span class="ko">모든 트랙을 합친 최신 글 목록입니다.</span>
      </p>
    </div>

    <div class="entry-list">
      {% for post in site.posts limit: 6 %}
        {% assign post_category_slug = post.categories | first %}
        {% assign post_category = site.data.categories | where: "slug", post_category_slug | first %}
        <article class="entry-row">
          <div class="entry-date bilingual">
            <span class="en">{{ post.date | date: "%Y-%m-%d" }}</span>
            <span class="ko">{{ post.date | date: "%Y-%m-%d" }}</span>
          </div>

          <div class="entry-body">
            <div class="entry-meta">
              {% if post_category %}
                <a class="doc-tag bilingual inline" href="{{ post_category.path | relative_url }}">
                  <span class="en">{{ post_category.en }}</span>
                  <span class="ko">{{ post_category.kr }}</span>
                </a>
              {% endif %}
            </div>

            <h3 class="entry-title">
              <a href="{{ post.url | relative_url }}">
                <span class="bilingual">
                  <span class="en">{{ post.title }}</span>
                  {% if post.title_kr %}
                    <span class="ko">{{ post.title_kr }}</span>
                  {% endif %}
                </span>
              </a>
            </h3>

            <p class="entry-excerpt bilingual">
              <span class="en">{{ post.excerpt_en | default: post.excerpt | strip_html | strip_newlines }}</span>
              <span class="ko">{{ post.excerpt_kr | default: post.excerpt | strip_html | strip_newlines }}</span>
            </p>
          </div>
        </article>
      {% endfor %}
    </div>
  </section>
</section>
