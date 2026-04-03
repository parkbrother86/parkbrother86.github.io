---
title: Home
---

<section class="panel category-board">
  <header class="category-board-head">
    <span class="eyebrow bilingual">
      <span class="en">Latest by Category</span>
      <span class="ko">카테고리별 최신 글</span>
    </span>
  </header>

  <div class="category-list">
    {% for category in site.data.categories %}
      {% assign latest_post = site.categories[category.slug] | first %}
      <article class="category-row">
        <div class="category-summary">
          <span class="category-index">{{ forloop.index | prepend: '0' }}</span>
          <a class="category-name-link" href="{{ category.path | relative_url }}">
            <h2 class="category-name bilingual">
              <span class="en">{{ category.en }}</span>
              <span class="ko">{{ category.kr }}</span>
            </h2>
          </a>
        </div>

        <div class="category-entry">
          {% if latest_post %}
            <a class="category-entry-title" href="{{ latest_post.url | relative_url }}">
              <span class="bilingual">
                <span class="en">{{ latest_post.title }}</span>
                {% if latest_post.title_kr %}
                  <span class="ko">{{ latest_post.title_kr }}</span>
                {% endif %}
              </span>
            </a>
            <span class="category-entry-date bilingual">
              <span class="en">{{ latest_post.date | date: "%B %d, %Y" }}</span>
              <span class="ko">{{ latest_post.date | date: "%Y-%m-%d" }}</span>
            </span>
            <a class="category-browse bilingual inline" href="{{ category.path | relative_url }}">
              <span class="en">Browse category</span>
              <span class="ko">카테고리 보기</span>
            </a>
          {% else %}
            <span class="category-entry-empty bilingual">
              <span class="en">No posts yet.</span>
              <span class="ko">아직 글이 없습니다.</span>
            </span>
            <a class="category-browse bilingual inline" href="{{ category.path | relative_url }}">
              <span class="en">Open category</span>
              <span class="ko">카테고리 열기</span>
            </a>
          {% endif %}
        </div>
      </article>
    {% endfor %}
  </div>
</section>
