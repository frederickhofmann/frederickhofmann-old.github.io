---
layout: page
title: Design
description: "Design Blog by Haufe Group"
navigation_weight: 3
permalink: /design/
---

{% for post in paginator.posts %}
<div class="post-preview">
    <a href="{{ post.url | prepend: site.baseurl }}">
        <h2 class="post-title">            {{ post.title }}
        </h2>
        {% if post.subtitle %}
        <h3 class="post-subtitle">
            {{ post.subtitle }}
        </h3>
        {% endif %}
    </a>
    {% if post.author %}
      {% assign author = site.data.authors[post.author] %}
      {% if author %}
        {% assign author_name = author.name %}
      {% else %}
        {% assign author_name = post.author %}
      {% endif %}
    {% else %}
      {% assign author_name = site.title %}
    {% endif %}
    <p class="post-meta">Posted by {{ author_name }} on {{ post.date | date: "%B %-d, %Y" }}</p>
</div>
<hr>
{% endfor %}

<!-- Pager -->
{% if paginator.total_pages > 1 %}
<ul class="pager">
    {% if paginator.previous_page %}
    <li class="previous">
        <a href="{{ paginator.previous_page_path | prepend: site.baseurl | replace: '//', '/' }}">&larr; Newer Posts</a>
    </li>
    {% endif %}
    {% if paginator.next_page %}
    <li class="next">
        <a href="{{ paginator.next_page_path | prepend: site.baseurl | replace: '//', '/' }}">Older Posts &rarr;</a>
    </li>
    {% endif %}
</ul>
{% endif %}