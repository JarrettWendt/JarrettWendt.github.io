---
title: Blog
layout: default
category: top
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

		</div>
	</div>
</div>
