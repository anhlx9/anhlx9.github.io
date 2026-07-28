---
title: "All Posts"
layout: archive
permalink: /all-posts/
author_profile: true
---

<p class="allposts-count">{{ site.posts | size }} bài viết — mới nhất trước.</p>

{% for post in site.posts %}
  {% include archive-single.html %}
{% endfor %}
