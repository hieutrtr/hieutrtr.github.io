---
layout: default
title: Kỹ thuật & AI đúng cách
lang: vi
ref: home
permalink: /vi/
---

{% include lang-switcher.html %}

Chào mừng bạn đến với **Kỹ thuật & AI đúng cách** — nhật ký tập trung vào phương pháp kỹ thuật chuyên nghiệp, kỷ luật và cách vận dụng AI một cách thực dụng.

Tại đây bạn sẽ tìm thấy:
- Hướng dẫn thực tế để xây dựng hệ thống dễ bảo trì.
- Chiến lược áp dụng công cụ AI nhằm mang lại giải pháp tin cậy, chất lượng cao.
- Góc nhìn về giao điểm giữa công nghệ, con người và quy trình.

Đón xem các bài phân tích chuyên sâu, hướng dẫn và quan điểm giúp đội ngũ đưa kỹ thuật và AI vào công việc một cách chuẩn mực.

## Bài viết mới nhất

{% assign localized_posts = site.posts | where: "lang", page.lang %}
{% if localized_posts.size > 0 %}
<ul class="post-list">
  {% for post in localized_posts %}
    <li>
      <span class="post-meta">{{ post.date | date: "%d/%m/%Y" }}</span>
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
<p>Chưa có bài viết nào.</p>
{% endif %}
