---
layout: default
---

<!-- MathJax JavaScript LaTeX library -->
<script src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-AMS-MML_HTMLorMML" type="text/javascript"></script>

<article class="post h-entry" itemscope itemtype="http://schema.org/BlogPosting">

	<header class="post-header">
		<h1 class="post-title p-name" itemprop="name headline">{{ page.title | escape }}</h1>
	</header>

	<div class="post-content e-content" itemprop="articleBody">
		{% if page.sourceRepo %}
			You can find all the source code in this article on my <a href="{{ page.sourceRepo }}" target="_blank">repo</a>.
			<br/><br/>
		{% endif %}

		{{ content }}

		{% if page.sourceRepo %}
			<sub>You can find all the source code in this article on my <a href="{{ page.sourceRepo }}" target="_blank">repo</a>.</sub>
		{% endif %}
	</div>

	<p class="post-meta">
		{%- assign date_format = site.minima.date_format | default: "%b %-d, %Y" -%}
		{%- if page.modified_date -%}
			{%- assign mdate = page.modified_date | date_to_xmlschema -%}
			<time class="dt-modified" datetime="{{ mdate }}" itemprop="dateModified">
				Last Updated on {{ mdate | date: date_format }}
			</time>
		{%- else -%}
			<time class="dt-published" datetime="{{ page.date | date_to_xmlschema }}" itemprop="datePublished">
				{{ page.date | date: date_format }}
			</time>
		{%- endif -%} 
		{%- if page.author -%}
			• {% for author in page.author %}
				<span itemprop="author" itemscope itemtype="http://schema.org/Person">
					<span class="p-author h-card" itemprop="name">{{ author }}</span></span>
					{%- if forloop.last == false %}, {% endif -%}
			{% endfor %}
		{%- endif -%}
	</p>
	

	<a class="u-url" href="{{ page.url | relative_url }}" hidden></a>
</article>


{%- if site.hyvor_talk_website_id -%}
	<div class="comments">
		{%- include hyvor-talk-comments.html -%}
	</div>
{%- endif -%}
