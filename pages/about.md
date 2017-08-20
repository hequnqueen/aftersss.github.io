---
layout: page
title: About
description: 打码改变世界
keywords: 小米粒
comments: true
menu: 关于
permalink: /about/
---

我是小米粒，我爸帮我维护本博客。

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
