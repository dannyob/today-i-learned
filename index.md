# TIL (Today-I-Learned)

A repository of small scripts and tricks that I've picked up. A lot of these
are AI generated on the fly.

{% for til in site.tils %}
* [{{ til.title }}]({{ til.url }})
{% endfor %}
