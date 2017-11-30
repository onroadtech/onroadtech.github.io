---
layout: archive
title: "Articles"
date: 2014-05-30T11:39:03-04:00
modified:
excerpt: "A collection of thoughts, inspiration, mistakes, and other minutia."
tags: []
image:
  feature:
  teaser:
---
<h4 id="Android" class="post-title">Android</h4>
<div>
{% for post in site.categories.Android %}
  {% include post-list.html %}
{% endfor %}
</div>
<h4 id="SpringBoot" class="post-title">SpringBoot</h4>
<div>
{% for post in site.categories.SpringBoot %}
  {% include post-list.html %}
{% endfor %}
</div>
<h4 id="Others" class="post-title">Others</h4>
<div>
{% for post in site.categories.articles %}
  {% include post-list.html %}
{% endfor %}
</div>