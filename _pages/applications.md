---
layout: page
title: applications
permalink: /applications/
description: 실생활에 유용한 앱을 개발합니다.
nav: true
nav_order: 4
display_categories: [경제]
horizontal: false
---

<!-- pages/applications.md -->
<div class="projects">
{% if site.enable_project_categories and page.display_categories %}
  <!-- Display categorized applications -->
  {% for category in page.display_categories %}
  <a id="{{ category }}" href=".#{{ category }}">
    <h2 class="category">{{ category }}</h2>
  </a>
  {% assign categorized_applications = site.applications | where: "category", category %}
  {% assign sorted_applications = categorized_applications | sort: "importance" %}
  <!-- Generate cards for each project -->
  {% if page.horizontal %}
  <div class="container">
    <div class="row row-cols-1 row-cols-md-2">
    {% for application in sorted_applications %}
      {% include applications_horizontal.liquid %}
    {% endfor %}
    </div>
  </div>
  {% else %}
  <div class="row row-cols-1 row-cols-md-3">
    {% for application in sorted_applications %}
      {% include applications.liquid %}
    {% endfor %}
  </div>
  {% endif %}
  {% endfor %}

{% else %}

<!-- Display applications without categories -->

{% assign sorted_applications = site.applications | sort: "importance" %}

  <!-- Generate cards for each project -->

{% if page.horizontal %}

  <div class="container">
    <div class="row row-cols-1 row-cols-md-2">
    {% for application in sorted_applications %}
      {% include applications_horizontal.liquid %}
    {% endfor %}
    </div>
  </div>
  {% else %}
  <div class="row row-cols-1 row-cols-md-3">
    {% for application in sorted_applications %}
      {% include applications.liquid %}
    {% endfor %}
  </div>
  {% endif %}
{% endif %}
</div>
