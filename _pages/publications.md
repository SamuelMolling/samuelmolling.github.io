---
layout: archive
title: "Publications"
permalink: /publications/
author_profile: true
---

{% if site.author.googlescholar %}
  <div class="wordwrap">You can also find my articles on <a href="{{site.author.googlescholar}}">my Google Scholar profile</a>.</div>
{% endif %}

{% include base_path %}

<!-- Filtros -->
<div id="filters">
  <button class="filter-button" data-tag="">All</button>
  {% assign tags = site.publications | map: 'tags' | flatten | uniq %}
  {% for tag in tags %}
    <button class="filter-button" data-tag="{{ tag }}">{{ tag }}</button>
  {% endfor %}
</div>

<!-- Publicações -->
<div id="posts">
  {% for post in site.publications reversed %}
    <div class="post" data-tags="{{ post.tags | join: ' ' }}">
      {% include archive-single.html %}
    </div>
  {% endfor %}
</div>

<script>
  document.addEventListener('DOMContentLoaded', () => {
    const filters = document.querySelectorAll('#filters .filter-button');
    const posts = document.querySelectorAll('#posts .post');

    filters.forEach(button => {
      button.addEventListener('click', () => {
        const tag = button.dataset.tag;
        posts.forEach(post => {
          if (post.dataset.tags.includes(tag) || tag === '') {
            post.style.display = 'block';
          } else {
            post.style.display = 'none';
          }
        });
      });
    });
  });
</script>

<style>
  #filters {
    margin-bottom: 20px;
  }

  .filter-button {
    margin: 5px;
    padding: 5px;
    border: none;
    border-bottom: 2px solid white;
    background-color: transparent;
    color: white;
    cursor: pointer;
    transition: color 0.3s ease, border-color 0.3s ease;
  }

  .filter-button:hover {
    color: #d5d9f7;
    border-bottom: 2px solid #d5d9f7;
  }

  .filter-button:focus {
    outline: none;
  }
</style>
