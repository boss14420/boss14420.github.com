---
layout: page
title: Boss14420's jekyll blog
---
{% include JB/setup %}

{% for post in site.posts limit:5 %}
#[{{ post.title }}]({{ post.url }})
{{ post.content }}
*****
{% endfor %}

## To-Do

This theme is still unfinished. If you'd like to be added as a contributor, [please fork](http://github.com/plusjade/jekyll-bootstrap)!
We need to clean up the themes, make theme usage guides with theme-specific markup examples.


