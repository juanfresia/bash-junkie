---
layout: default
title: Home
---

# Greetings

Welcome, traveller, to the Bash Junkie secret post stash. Suit yourself, you may
learn something new!

## Latest posts

{% for post in site.posts %}
### [{{ post.title }}]({{ post.url | relative_url }})
{% endfor %}
