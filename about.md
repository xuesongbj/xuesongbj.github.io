---
layout: page
title: "关于"
permalink: about.html
image: /public/images/flag.jpg
color: '#f44336'
sequence: 9
---


{% comment %}
  This inserts the "about" photo and text from `_config.yml`.
  You can edit it there (jekyll needs restart!) or remove it and provide your own photo/text.
  Don't forget to add the `me` class to the photo, like this: `![alt](src){:.me}`.
{% endcomment %}

{% if site.author.photo %}
  ![{{site.author.name}}]({{site.author.photo}}){:.me}
{% endif %}


我是<u>雪松</u>，毕业于北京交通大学。我目前在中国北京居住和工作，从事 IT 行业。我喜爱阅读、足球、数码技术和一切美的事物，也喜欢<del>哲学</del>和历史。


## 更加了解我

我在 [GitHub](https://github.com/myanbin) 上维护我的代码以及关注开源项目。


## 为什么要写这个博客？

花时间进行写作是一件很有意义也很值得去做的事。我希望能在这里来分享技术、记录生活，同时也希望能结交到更多朋友。

## 联系我

如果需要私下联系我，请发邮件到 [{{site.author.email}}](mailto:{{site.author.email}})。
