---
layout: default
title: writing
---

# writing

thoughts on applied AI, infrastructure, and building real systems

<ul>
  {% for post in site.posts %}
    <li>
      <a href="{{ post.url | relative_url }}">{{ post.title }}</a>
      <!-- <span> – {{ post.date | date: "%Y-%m-%d" }}</span>-->
      <br/>
      <small>{{ post.description }}</small> 
    </li>
  {% endfor %}
</ul>
