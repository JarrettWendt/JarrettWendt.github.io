---
title: Blog
layout: default
category: top
cwd: '../'
---

<div class="home">
	<div class="box home-box">
		<div class="post-list">

			{% assign page = site.pages | where: 'title', 'Making a Game Engine from Scratch' | first %}
			
			<a class="post-link" href="{{ page.url | relative_url }}">
				<img src="{{ page.thumb }}" alt="{{ page.thumb }}" height="100" style="float: right; vertical-align: top;">
				<h3>{{ page.title | escape }}</h3>
				<p>{{ page.excerpt }}</p>
				<sub><sub>2020 • Jarrett Wendt</sub></sub>
			</a>

			{% for post in site.misc_blog %}
				<a class="post-link" href="{{ post.url | relative_url }}">
					{% if post.thumb %}
						<img src="{{ post.cwd }}{{ post.thumb }}" alt="{{ post.thumb }}" height="100" style="float: right; vertical-align: top;">
					{% endif %}
					<h3>{{ post.title | escape }}</h3>
					<p>{{ post.excerpt }}</p>
					<sub><sub>{{ post.date | date_to_string }} • Jarrett Wendt</sub></sub>
				</a>
			{% endfor %}

		</div>
	</div>
</div>
