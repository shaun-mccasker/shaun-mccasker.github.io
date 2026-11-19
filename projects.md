---
layout: page
permalink: /projects/
---

<h1>Personal Projects</h1>

A collection of Personal Projects

<div class="writeup-grid">

  {% assign projects = site.projects | sort: "date" | reverse %}

  {% for project in projects %}

    <a href="{{ project.url | relative_url }}" class="writeup-card">

      <div class="writeup-meta">
        <span>{{ project.platform }}</span>
        <span>{{ project.difficulty }}</span>
      </div>

      <h3>{{ project.title }}</h3>

      <p>{{ project.description }}</p>

      <div class="writeup-footer">
        <span>{{ project.category }}</span>
        <span>Read project →</span>
      </div>

    </a>

  {% endfor %}

</div>