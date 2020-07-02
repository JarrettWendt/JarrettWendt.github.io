---
layout: default
title: Making a Game Engine from Scratch
excerpt: No, not <i>that</i> Scratch...
author: Jarrett Wendt
year: 2020
cwd: '../'
---

<div class="home">

	{% assign posts = site.posts %}
	
	<div class="box home-box">
		<div class="post-list">
			<div class="post-category">
				<h1>{{ page.title }}</h1>
				This blog series is about my journey creating a custom game engine from nothing. I'll be drawing inspiration from some of the most popular game engines freely available: Unreal, Unity, and Godot. For this project, I'll be using the latest versions of C++ and Visual Studio with a target platform of 64-bit Windows and Linux.
				<br/><br/>

				The primary focus of this project is to design the logical layout of data and control flow within the engine. This is the foundation that must be established before more obvious components of a game engine like physics and rendering can be added. So I'll be covering things like memory management, data structures, serialization, events, threading, and scripting language integration.
			</div>

			{% assign posts = site.custom_engine | sort: 'date' %}
			{%- for post in posts -%}
				<a class="post-link" href="{{ post.url | relative_url }}">
					{% if post.thumb %}
						<img src="../{{ post.thumb }}" alt="{{ post.thumb }}" height="100" style="float: right; vertical-align: top;">
					{% endif %}
					<h3>{{ post.title | escape }}</h3>
					<p>{{ post.excerpt }}</p>
					<sub><sub>{{ post.date | date_to_string }} • {{ post.author }}</sub></sub>
				</a>
			{%- endfor -%}
		</div>
	</div>

</div>
