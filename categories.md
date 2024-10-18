---
layout: page
title: Categories
permalink: /categories/
---

A breakdown of the categories within the various blog posts.

{% for category in site.categories %}
  <H3>{{ category[0] }}</H3>
  <UL>
    {% for post in category[1] %}
      <LI><A HREF="{{ post.url }}">{{ post.title }}</A></LI>
    {% endfor %}
  </UL>
{% endfor %}
