---
layout: default
title: Making a Game Engine from Scratch
excerpt: No, not <i>that</i> Scratch...
author: Jarrett Wendt
cwd: '../'
---

<div class="home">

	{% assign posts = site.posts %}
	
	<div class="box home-box">
		<div class="post-list">
			<div class="post-category">
			{% markdown %}
# {{ page.title }}

This blog series is about my journey creating a custom game engine from nothing. I'll be drawing inspiration from the two most popular game engines freely available: Unity and Unreal. For this project, I'll be using the latest versions of C++ and Visual Studio with a target platform of Windows x64.

The primary focus of this project is to design the logical layout of data and control flow within the engine. This is the foundation that needs to be established before things like physics and rendering can be added. So I'll be covering things like data memory management, data structures, serialization, events, threading, and scripting language integration.

			{% endmarkdown %}
			</div>

			{% assign posts = site.posts | sort: 'date' %}
			{%- for post in posts -%}
				{% if post.category == "customEngine" %}
					<a class="post-link" href="{{ post.url | relative_url }}">
						<img src="../{{ post.thumb }}" alt="{{ post.thumb }}" height="100" style="float: right; vertical-align: top;">
						<h3>{{ post.title | escape }}</h3>
						<p>{{ post.excerpt }}</p>
						<sub><sub>{{ post.date | date_to_string }} • {{ post.author }}</sub></sub>
					</a>
				{% endif %}
			{%- endfor -%}
		</div>
	</div>

</div>
