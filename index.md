---
layout: default
---

{% for post in site.posts %}
<span class="idx-post">`{{ post.date | date: "%b %d %Y" }}`</span> [{{ post.title }}]({{ post.url }})
{% endfor %}