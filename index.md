---
layout: default
---

A repository of small scripts and tricks that I've picked up. A lot of these
are AI-generated tools, and transcripts of LLM conversations.

{% for til in site.tils %}
* {{til.date | date: "%Y-%m-%d"}} [{{ til.title }}]({{ til.url }}) 
{% endfor %}
