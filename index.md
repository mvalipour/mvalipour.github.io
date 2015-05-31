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
			<h1 class="title-field">
        <a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a>
      </h1>
      <div class='blog-post-details'>
      	<span>
      		<span class="fa fa-calendar"></span>
      		{{ post.date | date_to_long_string }}
      	</span>
        &nbsp;
        <span class="fa fa-comments"></span>
        <a href="{{ BASE_PATH }}{{ post.url }}#disqus_thread">Comments</a>
        &nbsp;
      	{% unless post.tags == empty %}
      	<span>
      		<span class="fa fa-tags"></span>
      		{% assign tags_list = post.tags %}
      		{% include JB/tags_list %}
      	</span>
      	{% endunless %}
      </div>
			<div class="summary-field">
				{% if post.content contains '<!--more-->' %}
				  {{ post.content | split:'<!--more-->' | first }}
				{% else %}
				  {{ post.content }}
				{% endif %}
			</div>
			<div class="buttons">
				<a href="{{ BASE_PATH }}{{ post.url }}">Read more &raquo;</a>
			</div>
		</div>
	</div>
  {% endfor %}
</div>
