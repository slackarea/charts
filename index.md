---
layout: default
---

# Benvenuto nel mio spazio digitale

Questo è il mio sito personale dove condivido progetti, pensieri e scoperte nel mondo della tecnologia.

## Ultimi Progetti

- [Progetto 1](link-progetto-1) - Descrizione breve
- [Progetto 2](link-progetto-2) - Descrizione breve
- [Progetto 3](link-progetto-3) - Descrizione breve

## Ultimi Post

{% for post in site.posts limit:3 %}
- [{{ post.title }}]({{ post.url }}) - {{ post.date | date: "%d %B %Y" }}
{% endfor %}

[Vedi tutti i post](/blog/)
