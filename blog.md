---
title: Blog
layout: default
---

<div class="home">

	{% assign posts = site.posts %}

	{%- if posts.size > 0 -%}
		{%- if page.list_title -%}
			<h2 class="post-list-heading">{{ page.list_title }}</h2>
		{%- endif -%}
		<div class="box home-box">
			<div class="post-list">
				{%- for post in posts -%}
					<a class="post-link" href="{{ post.url | relative_url }}">
						<h3>{{ post.title | escape }}</h3>
						<p>{{ post.excerpt }}</p>
						<sub><sub>{{ post.date | date_to_string }} • {{ post.author }}</sub></sub>
					</a>
				{%- endfor -%}
			</div>
		</div>

	{%- endif -%}

</div>
