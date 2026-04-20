---
layout: page
title: LLM Interp
permalink: /llm-interp/
---

Mechanistic interpretability notes on large language models — steering
vectors, probes, and activation-level experiments.

<ul class="post-list">
  {% assign posts = site.categories['llm-interp'] %}
  {% for post in posts %}
    <li>
      <span class="post-meta">{{ post.date | date: "%b %-d, %Y" }}</span>
      <h3>
        <a class="post-link" href="{{ post.url | relative_url }}">{{ post.title | escape }}</a>
      </h3>
      {% if post.excerpt %}
        <p>{{ post.excerpt | strip_html | truncate: 240 }}</p>
      {% endif %}
    </li>
  {% endfor %}
</ul>
