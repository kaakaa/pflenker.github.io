---
layout: page
title: "Product Owner Q&A"
excluded: true
image: /assets/teaching.png
excerpt: "Questions around day to day Product Owner work, asked and answered by Product Owners and me."
---
As part of a long-running Product Owner training I hosted, I was offering a Lean Coffee where every participant could ask questions and bring challenges, which would then be discussed and answered by the full group.

Here, I will collect some interesting questions that have arisen, as well as the answers we found.

{% for po_qa in site.po_qa %}
{% if po_qa.excluded != true %}
### {{po_qa.title}}
{{po_qa.excerpt}}  
[&raquo;  Read more]({{po_qa.url}})
{% endif %}
{% endfor %}
