---
layout: default
---

主页也不知道写点啥，想到了再写，嘿嘿

文章列表：
<ul>
  {% for post in site.posts %}
    <li>
      {{page.category}} <h3><a href="{{ post.url }}">{{ post.title }}</a></h3> {{page.date}}
      <p>{{ post.excerpt }}</p>
    </li>
  {% endfor %}
</ul>


![Octocat](https://github.githubassets.com/images/icons/emoji/octocat.png)