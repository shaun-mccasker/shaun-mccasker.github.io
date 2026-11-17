---
layout: page
permalink: /ctf-writeups/
---

<h1>CTF Writeups</h1>

A collection of CTF challenges, security research and technical walkthroughs.

<div class="writeup-grid">

  {% assign writeups = site.writeups | sort: "date" | reverse %}

  {% for writeup in writeups %}

    <a href="{{ writeup.url | relative_url }}" class="writeup-card">

      <div class="writeup-meta">
        <span>{{ writeup.platform }}</span>
        <span>{{ writeup.difficulty }}</span>
      </div>

      <h3>{{ writeup.title }}</h3>

      <p>{{ writeup.description }}</p>

      <div class="writeup-footer">
        <span>{{ writeup.category }}</span>
        <span>Read writeup →</span>
      </div>

    </a>

  {% endfor %}

</div>