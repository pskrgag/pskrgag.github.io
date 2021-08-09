---
layout: default
permalink: /blog
permalink_name: /blog
title: Blog
---

<ul>
	{% for post in site.posts %}
	  <li>
	    <a href="{{ post.url }}">{{ post.title }} - {{ post.date | date_to_long_string }}</a>
	  </li>
	{% endfor %}
</ul>