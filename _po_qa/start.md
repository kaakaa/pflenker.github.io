---
layout: post
title: "Product Owner Q&A"
excluded: true
image: /assets/teaching.png
---
One part of my job is coaching Product Owners, Product Managers and people 
in other roles who are interested in Product Owner related topics. I am currently offering a weekly Lean Coffee, where everyone can bring questions and challenges, 
and we as a group discuss them.

Here, I will collect some interesting questions that have arisen on these occurences, as well as the answers we found:

{% for po_qa in site.po_qa %}
{% if po_qa.excluded != true %}
- [{{po_qa.title}}]({{po_qa.url}})
{% endif %}
{% endfor %}
