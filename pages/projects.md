---
title: Projects
layout: default
category: top
cwd: '../'
---

<div class="home">

	{% assign posts = site.posts %}
	
	<div class="box home-box">
		<div class="post-list">
			<div class="post-category">
				<h1>{{ page.title }}</h1>
				First paragraph
				<br/><br/>

				Second paragraph
			</div>			

			{% assign posts = site.pages | where: 'title', 'Making a Game Engine from Scratch' | concat: site.projects | sort: 'date' %}
			{%- for post in posts -%}
				<a class="post-link" href="{{ post.url | relative_url }}">
					<img src="../{{ post.thumb }}" alt="{{ post.thumb }}" height="100" style="float: right; vertical-align: top;">
					<h3>{{ post.title | escape }}</h3>
					<p>{{ post.excerpt }}</p>
					<sub><sub>{{ post.date | date_to_string }} • {{ post.author }}</sub></sub>
				</a>
			{%- endfor -%}
		</div>
	</div>

</div>
