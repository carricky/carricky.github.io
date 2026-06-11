---
layout: page
permalink: /photos/
title: photos
description: A few favorites from my 500px gallery. Click any frame to view it on 500px.
nav: true
nav_order: 3
---

{% assign photos = site.data.photos500px.photos %}

{% if photos and photos.size > 0 %}

<!-- Auto-generated from https://500px.com/carricky via the 500px API at build
     time; a daily scheduled rebuild keeps this in sync with new uploads. -->

<div class="row">
  {% for photo in photos limit: 9 %}
    {% comment %} pick the 440px thumbnail, fall back to the first image {% endcomment %}
    {% assign thumb = "" %}
    {% for img in photo.images %}
      {% if img.size == 440 %}{% assign thumb = img.https_url | default: img.url %}{% endif %}
    {% endfor %}
    {% if thumb == "" %}{% assign thumb = photo.images.first.https_url | default: photo.images.first.url %}{% endif %}
    <div class="col-sm-4 mb-4">
      <a href="https://500px.com{{ photo.url }}" target="_blank" rel="noopener" title="{{ photo.name | escape }}">
        <img class="img-fluid rounded z-depth-1" loading="lazy" src="{{ thumb }}" alt="{{ photo.name | escape }}">
      </a>
    </div>
  {% endfor %}
</div>

<p class="text-center mt-3">See the full collection on my <a href="https://500px.com/carricky" target="_blank" rel="noopener">500px gallery</a>.</p>

{% else %}

<!-- Fallback shown if the 500px feed is unavailable at build time. -->

<div class="row">
  <div class="col-sm mt-3 mt-md-0">
    {% include figure.liquid loading="eager" path="assets/img/in-post/15319693174453/20180612-GSY_4934.jpg" class="img-fluid rounded z-depth-1" zoomable=true caption="Sunset over East Rock, New Haven" %}
  </div>
</div>

For the full collection, head over to my [500px gallery](https://500px.com/carricky).

{% endif %}
