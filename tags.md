---
layout: default
title: 태그
description: 주제별로 모아 보기
permalink: /tags/
---
<style>
  .tag-cloud { display:flex; flex-wrap:wrap; gap:8px; margin:0 0 32px; }
  .tag-chip { display:inline-flex; align-items:center; gap:6px; padding:5px 12px; border-radius:99px; background:#f2eef7; color:#5F0080; text-decoration:none; font-size:14px; font-weight:600; border:1px solid #e4dcee; }
  .tag-chip:hover { background:#5F0080; color:#fff; }
  .tag-chip .cnt { font-size:12px; opacity:.7; font-variant-numeric:tabular-nums; }
  .tag-section { margin:0 0 28px; scroll-margin-top:20px; }
  .tag-section h2 { font-size:18px; color:#5F0080; margin:0 0 8px; padding-bottom:6px; border-bottom:1px solid #ece8f2; }
  .tag-section ul { list-style:none; padding:0; margin:0; display:flex; flex-direction:column; gap:6px; }
  .tag-section li { display:flex; align-items:baseline; justify-content:space-between; gap:12px; }
  .tag-section a { color:#2a2433; text-decoration:none; border-bottom:1px solid transparent; }
  .tag-section a:hover { color:#5F0080; border-bottom-color:#5F0080; }
  .tag-date { flex:none; font-size:12px; color:#a49dad; font-variant-numeric:tabular-nums; }
  :root[data-theme="dark"] .tag-chip { background:#2a2533; color:#c9a8e6; border-color:#3a3348; }
  :root[data-theme="dark"] .tag-chip:hover { background:#5F0080; color:#fff; }
  :root[data-theme="dark"] .tag-section h2 { border-color:#2f2838; }
  :root[data-theme="dark"] .tag-section a { color:#e6e0ef; }
</style>

<h1>🏷️ 태그</h1>
<p style="color:#6a6472;margin:0 0 22px;">관심 주제로 모아 보세요. 숫자는 글 수입니다.</p>

{% assign sorted = site.tags | sort %}
<div class="tag-cloud">
  {% for tag in sorted %}
  <a class="tag-chip" href="#{{ tag[0] | slugify }}">{{ tag[0] }} <span class="cnt">{{ tag[1].size }}</span></a>
  {% endfor %}
</div>

{% for tag in sorted %}
<section class="tag-section" id="{{ tag[0] | slugify }}">
  <h2>{{ tag[0] }}</h2>
  <ul>
    {% for p in tag[1] %}
    <li><a href="{{ p.url | relative_url }}">{{ p.title }}</a><span class="tag-date">{{ p.date | date: '%Y-%m-%d' }}</span></li>
    {% endfor %}
  </ul>
</section>
{% endfor %}
