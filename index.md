---
title: Dererif
hide_title: true
---

## Recent Posts

<ul class="post-list">
  {% for post in site.posts limit: 5 %}
    {% include post_list_item.html post=post show_categories=true %}
  {% endfor %}
</ul>

<p><a href="{{ '/topics/' | relative_url }}">See every topic</a></p>
