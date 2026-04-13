---
title: Topics
permalink: /topics/
---

<div class="bilingual">
  <p class="en">Browse every writing track from one place. Each topic page groups related posts without adding extra layout complexity.</p>
  <p class="ko">모든 글 주제를 한 곳에서 볼 수 있습니다. 각 주제 페이지는 관련 글만 단순하게 묶어 보여줍니다.</p>
</div>

{% assign topic_pages = site.pages | where_exp: "item", "item.taxonomy" | sort: "title" %}

<ul class="topic-list">
  {% for topic in topic_pages %}
    {% assign post_count = site.categories[topic.taxonomy] | size %}
    <li>
      <h3><a href="{{ topic.url | relative_url }}">{{ topic.title }}</a></h3>
      <p class="post-meta">{{ post_count }} posts</p>
      <div class="bilingual">
        <p class="en">{{ topic.summary_en }}</p>
        <p class="ko">{{ topic.summary_kr }}</p>
      </div>
    </li>
  {% endfor %}
</ul>
