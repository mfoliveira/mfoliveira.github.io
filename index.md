---
title: hello world
---

# about

This is the blog of Mauricio Faria de Oliveira,  
mostly about adventures in GNU/Linux.

# blog

<ul>
{% for post in site.posts %}
	<li>{{ post.date | date_to_string }} :: <a href="{{ post.url }}">{{ post.title }}</a></li>
{% endfor %}
</ul>

# links

<ul>
	<li><a href="{{ site.baseurl }}/contributions">Open Source Contributions</a></li>
	<li>Email: mauricio.foliveira gmail com</li>
	<li>IRC: mfo @ irc.libera.chat, irc.oftc.net</li>
	<li><a href="https://github.com/mfoliveira/">GitHub</a></li>
	<li><a href="https://www.linkedin.com/in/mauriciofariadeoliveira/">LinkedIn</a></li>
	<li><a href="{{ '/feed.xml' | relative_url }}">RSS</a></li>
</ul>
