---
layout: default
title: Wiki 
image:
  path: https://avatars.githubusercontent.com/u/1961587?s=200&v=4
  height: 200
  width: 200
---

My personal tech-related wiki, mostly notes-to-self from my days as a sysadmin.

Note that these posts are written for myself and thus *are not written for public consumption*; but if anyone happens to find anything useful and have questions, let me know! I'd love to hear about it!

{% assign wikiPosts = site.pages | where: 'section', 'wikipost' %}
{% for post in wikiPosts %}
* <a href="{{ post.url }}">{{ post.title }}</a>
{% endfor %}
