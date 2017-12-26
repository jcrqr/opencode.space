---
layout: page
title: Random Posts
permalink: /categories/random
---

<div style="font-size: 130%; margin-bottom: 1.5rem">
  Random stuff.
</div>

<ul class="post-list">
  {% assign count = 0 %}

  {% for post in site.posts %}
    {% if post.categories contains "random" %}
      {% assign count = count | plus: 1 %}

      <li>
        {% assign date_format = site.minima.date_format | default: "%b %-d, %Y" %}
        <span class="post-meta">{{ post.date | date: date_format }}</span>

        <h2>
          <a class="post-link" href="{{ post.url | relative_url }}">{{ post.title | escape }}</a>
        </h2>
      </li>
    {% endif %}
  {% endfor %}

  {% if count == 0 %}
    <p>There aren't yet any posts in this category...</p>
  {% endif %}
</ul>
