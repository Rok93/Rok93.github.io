---
layout: default
title: 학습 지도
description: 무엇을 배웠고 무엇을 배울지 — 자동으로 자라는 학습 여정
---
{% assign total = 0 %}{% assign done = 0 %}
{% for c in site.data.roadmap.categories %}{% for it in c.items %}{% assign total = total | plus: 1 %}{% if it.done %}{% assign done = done | plus: 1 %}{% endif %}{% endfor %}{% endfor %}
{% assign pct = done | times: 100 | divided_by: total %}

<style>
  .rm-head { margin-bottom: 8px; }
  .rm-lede { color: #6a6472; margin: 0 0 20px; font-size: 15px; }
  .rm-bar-wrap { display:flex; align-items:center; gap:12px; margin: 0 0 34px; }
  .rm-bar { flex:1; height:12px; background:#ece8f2; border-radius:99px; overflow:hidden; }
  .rm-bar-fill { height:100%; background:linear-gradient(90deg,#5F0080,#9b4dca); border-radius:99px; }
  .rm-bar-num { font-variant-numeric: tabular-nums; font-weight:700; color:#5F0080; font-size:14px; white-space:nowrap; }
  .rm-cat { margin: 0 0 26px; }
  .rm-cat-name { font-size:13px; font-weight:700; letter-spacing:.04em; text-transform:uppercase; color:#8a7fa0; margin:0 0 10px; padding-bottom:6px; border-bottom:1px solid #ece8f2; }
  .rm-list { list-style:none; padding:0; margin:0; display:flex; flex-direction:column; gap:2px; }
  .rm-item { display:flex; align-items:baseline; gap:10px; padding:7px 8px; border-radius:8px; }
  .rm-item.done:hover { background:#faf8fc; }
  .rm-mark { flex:none; width:18px; text-align:center; font-size:13px; }
  .rm-item.done .rm-mark { color:#1f7a4d; }
  .rm-item.todo .rm-mark { color:#c9c2d6; }
  .rm-item.todo .rm-title { color:#a49dad; }
  .rm-item.done .rm-title a { color:#2a2433; text-decoration:none; border-bottom:1px solid transparent; }
  .rm-item.done .rm-title a:hover { border-bottom-color:#5F0080; color:#5F0080; }
  .rm-next { font-size:11px; font-weight:700; color:#fff; background:#5F0080; padding:1px 7px; border-radius:99px; margin-left:2px; }
  :root[data-theme="dark"] .rm-bar { background:#2a2533; }
  :root[data-theme="dark"] .rm-cat-name { color:#a99fc0; border-color:#2a2533; }
  :root[data-theme="dark"] .rm-item.done:hover { background:#221d2b; }
  :root[data-theme="dark"] .rm-item.done .rm-title a { color:#e6e0ef; }
  :root[data-theme="dark"] .rm-lede { color:#9990a6; }
</style>

<h1 class="rm-head">🗺️ 학습 지도</h1>
<p class="rm-lede">무엇을 배웠고, 무엇을 배울 차례인지. 이 블로그가 다루는 주제들이 자동으로 여기 쌓입니다.</p>

<div class="rm-bar-wrap">
  <div class="rm-bar"><div class="rm-bar-fill" style="width:{{ pct }}%"></div></div>
  <span class="rm-bar-num">{{ done }} / {{ total }} · {{ pct }}%</span>
</div>

{% assign firsttodo = true %}
{% for c in site.data.roadmap.categories %}
<div class="rm-cat">
  <div class="rm-cat-name">{{ c.name }}</div>
  <ul class="rm-list">
    {% for it in c.items %}
    <li class="rm-item {% if it.done %}done{% else %}todo{% endif %}">
      <span class="rm-mark">{% if it.done %}✓{% else %}○{% endif %}</span>
      <span class="rm-title">{% if it.done and it.url %}<a href="{{ it.url | relative_url }}">{{ it.title }}</a>{% else %}{{ it.title }}{% endif %}{% if it.done == false and firsttodo %}<span class="rm-next">다음</span>{% assign firsttodo = false %}{% endif %}</span>
    </li>
    {% endfor %}
  </ul>
</div>
{% endfor %}
