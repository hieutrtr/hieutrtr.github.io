---
layout: default
title: Engineering & AI the Right Way
lang: en
ref: home
---

Welcome to **Engineering & AI the Right Way**, a journal dedicated to professional, disciplined engineering practices and the pragmatic use of artificial intelligence.

Here you'll find:
- Practical guides for building maintainable systems.
- Strategies for applying AI tools to deliver reliable, high-quality solutions.
- Reflections on the intersection of technology, people, and process.

Stay tuned for deep dives, tutorials, and perspectives aimed at helping teams put engineering and AI to work the right way.

## Latest posts

{% assign localized_posts = site.posts | where: "lang", page.lang %}
{% if localized_posts.size > 0 %}
<ul class="post-list">
  {% for post in localized_posts %}
    <li>
      <span class="post-meta">{{ post.date | date: "%b %-d, %Y" }}</span>
      <h2>
        <a class="post-link" href="{{ post.url | relative_url }}">{{ post.title | escape }}</a>
      </h2>
      {% if post.description %}
        <p>{{ post.description }}</p>
      {% endif %}
    </li>
  {% endfor %}
</ul>
{% else %}
<p>No posts published yet.</p>
{% endif %}
