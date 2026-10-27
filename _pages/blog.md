---
layout: default
permalink: /blog/
title: Blog
nav: true
nav_order: 4
pagination:
  enabled: false
---

<div class="post">

  <header class="post-header">
    <h1 class="post-title">{{ site.blog_name }}</h1>
    <p class="post-description">{{ site.blog_description }}</p>
  </header>

  <ul class="post-list">
    {%- for post in site.posts -%}
    {%- assign read_time = post.content | number_of_words | divided_by: 180 | plus: 1 -%}
    {%- assign year = post.date | date: "%Y" -%}
    <li>
      <h3><a class="post-title" href="{{ post.url | relative_url }}">{{ post.title }}</a></h3>
      {%- if post.description %}
      <p class="post-description">{{ post.description }}</p>
      {%- endif %}
      <p class="post-meta">{{ post.date | date: "%B %-d, %Y" }} &nbsp; &middot; &nbsp; {{ read_time }} min read</p>
      <p class="post-tags">
        <a href="{{ year | prepend: '/blog/' | prepend: site.baseurl }}">
          <i class="fas fa-calendar fa-sm"></i> {{ year }}</a>
        {%- if post.tags.size > 0 %}
        &nbsp; &middot; &nbsp;
        {% for tag in post.tags -%}
        <a href="{{ tag | slugify | prepend: '/blog/tag/' | prepend: site.baseurl }}">
          <i class="fas fa-hashtag fa-sm"></i> {{ tag }}</a> &nbsp;
        {% endfor -%}
        {%- endif %}
      </p>
    </li>
    {%- endfor -%}
  </ul>

</div>
