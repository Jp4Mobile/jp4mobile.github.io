---
layout: page
title: Tags
permalink: /tags/
---

A breakdown of the tags within the various blog posts.

{% for tag in site.tags %}
  <H3>{{ tag[0] }}</H3>
  <UL>
    {% for post in tag[1] %}
      <LI><A HREF="{{ post.url }}">{{ post.title }}</A></LI>
    {% endfor %}
  </UL>
{% endfor %}
