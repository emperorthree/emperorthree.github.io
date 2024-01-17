---
layout: default
---

主页也不知道写点啥，想到了再写，嘿嘿

# 文章分类
<ul>
  {% for nav in site.labels %}
    <li>
        <a href="{{ site.url }}{{ nav.href }}" title="{{ nav.label }}" target="{{ nav.target | default: _self }}">{{ nav.label }}</a>
    </li>
    <ul>
    {% for post in site.posts %}
        {% if post.category==nav.label %}
        <li>
            <div>
                <span>{{ post.date | date: "%Y-%m-%d" }}</span>
                <a href="{{ site.url }}{{ post.url }}">{{ post.title }}</a>
            </div>
        </li>
        {% endif %}
    {% endfor %}
    </ul>
  {% endfor %}
</ul>

![Octocat](https://github.githubassets.com/images/icons/emoji/octocat.png)