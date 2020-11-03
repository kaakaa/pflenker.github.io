---
layout: page
title: Write-Ups
permalink: /write-ups/
image: /assets/typewriter.png
excerpt: "Write-ups around software engineering, leadership and more."
---
<p>
This is a collection of write-ups that I have published on <a href="/">the blog</a>. Usually, they revolve around how
to guide or lead Engineering Teams in an uncertain environment.  I also wrote a few bits about Product Management and Product Ownership, which <a href="/po_qa/start"> you can find here.</a>
</p>

<div class="posts">
  
   {% assign entries =  site.posts | where_exp: "item", "item.categories contains 'Write-Up'" %}
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
              <span class="post_date">{{ post.date | date: "%B %e, %Y" }}
              </span>
              {% if post.last_modified_at %}
              
              <span class="post_date" datetime="{{ post.last_modified_at | date_to_xmlschema }}">
                  &middot;
                  Updated: {{ post.last_modified_at | date: "%b %-d, %Y" }}
                </span>
              {% endif %}
              {%  if post.title != "" %}       
              <span class="reading_time">
               &middot; {% include reading_time.html content=post.content %}
              </span>
              {% endif %}
         
            </p>



          </div>
        </a>
      
      <div class="entry">
        {%  if post.title != "" %}
          {{ post.excerpt }}
        {%  else  %}
          {{ post.content }}
        {% endif %}  
      </div>

      <a href="{{ site.baseurl }}{{ post.url }}" class="read-more"> 
        {%  if post.title != "" %}
          &raquo;  Read More
        {%  else  %}
          &raquo;  View Post
        {% endif %}  
        
      </a>
    </article>
  {% endfor %}

 
</div>