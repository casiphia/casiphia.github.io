---
title: Home
layout: collection
permalink: /
entries_layout: grid
---

## 欢迎来到我的个人主页

这是我的个人网站。在这里你可以找到关于我的信息、我的博客文章，以及如何联系我。

## 关于我

我是一名软件开发者，对编程、技术和创新充满热情。我喜欢学习新的技术和分享我的知识。

[了解更多关于我的信息](/about)

## 最新博客文章

{% for post in site.posts %}

- [{{ post.title }}]({{ post.url }}) - {{ post.date | date: "%Y-%m-%d" }}
{% endfor %}

## 联系我

如果你有任何问题或者想要交流合作，请通过以下方式联系我：

- 电子邮件: [wang@gmail746277441.com](https://mail.google.com)
- GitHub: [https://github.com/casiphia](https://github.com/casiphia)

感谢你的访问！欢迎随时联系我。

---
