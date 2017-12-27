---
layout: page
title: About
description: 一个码农
keywords: 黑熊怪, 还是码农
comments: true
menu: 关于
permalink: /about/
---

## 座右铭

* 学海无涯苦作舟
* 三人行必有我师

## 联系

{% for website in site.data.social %}
* {{ website.sitename }}：[@{{ website.name }}]({{ website.url }})
{% endfor %}

## Skill Keywords

{% for category in site.data.skills %}
### {{ category.name }}
<div class="btn-inline">
{% for keyword in category.keywords %}
<button class="btn btn-outline" type="button">{{ keyword }}</button>
{% endfor %}
</div>
{% endfor %}
