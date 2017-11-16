---
layout: home
permalink: /
image:
  feature: wood-texture-1600x800.jpg
---

<h2 class="post-title">Latest Posts</h2>
<div class="tiles">
{% for post in site.posts limit:4 %}
	{% include post-grid.html %}
{% endfor %}
</div><!-- /.tiles -->

<div>
    <a href="{{ site.url }}/articles/" class="btn">All Post</a>
</div>