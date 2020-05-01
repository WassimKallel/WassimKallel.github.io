---
layout: page
title: aws
permalink: /blog/tags/aws
---
 
<h5> Posts by Tag : {{ page.title }} </h5>

<div class="card">
{% for post in site.tags.aws %}
 <li class="category-posts"><span>{{ post.date | date_to_string }}</span> &nbsp; <a href="{{ post.url }}">{{ post.title }}</a></li>
{% endfor %}
</div>