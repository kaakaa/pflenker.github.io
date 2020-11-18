---
layout: page
title: Social Media Postings
permalink: /insta/
excerpt: "Posts on Instagram and other social media"
---
>

<div class="posts">
  
   {% assign entries = site.insta | sort: "date" | reverse %}
    {% for post in entries %}
      <article class="post">
        <a href="{{ site.baseurl }}{{ post.url }}">
          {%  if post.title != "" %}
            <h1>{{ post.title }}</h1>
          {%  endif %}
          <div>
            {% if post.image %}
              <img src="{{site.baseurl}}{{post.image}}" alt="{{post.title}}"/>          
            {% endif %}  
        
            <p class="post_info">
              Posted on <span class="post_date">{{ post.date | date: "%B %e, %Y" }}
              </span>
              
            </p>



          </div>
        </a>
      
      <div class="entry">
          {{ post.excerpt }}
      </div>
    </article>
  {% endfor %}

 
</div>