---
layout: barepage
title: Welcome!
---
{% include JB/setup %}

<div class="home-page-posts">
  {% for post in site.posts %}
	{% capture theCycle %}{% cycle 'one', 'two', 'three', 'four', 'five, 'six' %}{% endcapture %}
	
	<div class="post-item alt-{{ theCycle }} index-{{ forloop.index }}">
	    <div class="container-narrow">
			<h2 class="title-field"><a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a> <small>&mdash; {{ post.date | date_to_string }}</small></h2>
			<div class="summary-field">
				{% if post.content contains '<!--more-->' %}
				  {{ post.content | split:'<!--more-->' | first }}
				{% else %}
				  {{ post.content }}
				{% endif %}
			</div>
			<div class="buttons">
				<a href="{{ BASE_PATH }}{{ post.url }}#comments">Comments &raquo;</a>
				•
				<a href="{{ BASE_PATH }}{{ post.url }}">Read more &raquo;</a>
			</div> 
		</div>
	</div>
  {% endfor %}
</div>

