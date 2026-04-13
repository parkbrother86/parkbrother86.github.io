---
title: Dererif
hide_title: true
---

<div class="bilingual">
  <p class="en">Notes on game server architecture, client engineering, debugging, and toolmaking. This version keeps the site intentionally simple on top of the Dinky theme.</p>
  <p class="ko">게임 서버 아키텍처, 클라이언트 엔지니어링, 디버깅, 도구 제작에 대한 기록입니다. 이번 구성은 Dinky 테마 위에서 최대한 단순하게 정리했습니다.</p>
</div>

<p class="action-row">
  <a class="buttons" href="{{ '/topics/' | relative_url }}">Browse Topics</a>
  <a class="buttons" href="{{ '/about/' | relative_url }}">About</a>
  <a class="buttons" href="{{ '/feed.xml' | relative_url }}">RSS Feed</a>
</p>

## Recent Posts

<ul class="post-list">
  {% for post in site.posts limit: 5 %}
    {% include post_list_item.html post=post show_categories=true %}
  {% endfor %}
</ul>

<p><a href="{{ '/topics/' | relative_url }}">See every topic</a></p>
