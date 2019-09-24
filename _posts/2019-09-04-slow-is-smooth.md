---
layout: post
title: Slow is smooth, and smooth is fast
categories: [Estimates] 
---
Today I stumbled over [this article](https://varnish-cache.org/docs/6.2/phk/thatslow.html) which makes a few excellent points about how fast (in terms of lines per person per hour) a developer usually is. I like that idea a lot, even though I would never try and use it to actually assess the speed of a developer. The underlying metric takes into account lines added and lines deleted as well, so measuring on lines added only would be meaningless.

The surprising result of the article is that a speed of five (5) lines per hour per person is not only normal, but also a bit on the high side. 

I wanted to know the numbers of the product I recently worked with. Obviously, the result would be very, very rough, but still I wanted to have at least some kind of indication.

First, I needed the number of lines of code. That was difficult to come by, as the code is spread across multiple repositories, and the repositories sometimes also contain documentation or tests. To keep things simple, I assumed that all the lines in all the repositories are code. I used a simple tool to count all lines per repository on github and arrived at 1.4 million lines of code.

Next, I needed the number of people working on the project. Again, this would be very rough, since the product is very mature, and over the years, a lot of developers came and went. There was one team working full time on the product, and another team half the time, and I estimated an average team size of 8 developers per team, so the assumption is that on average, 12 developers have worked on the product.

Third, I needed the number of hours which have went into the product. The product has been around for 15 years. It has greatly evolved over time, and there have been numerous rewrites, but since all the rewrites used the knowledge of prior rewrites to their advantages, I treated these rewrites as simple "remove/add" operations, same as the article proposes when you rewrite a function and throw out half the lines which have been in there before. We can also assume that the developers have worked around 7 hours per working day, and 200 days per year on the project.

Putting this all together, we arrive at 21,000 hours per person over 15 years, and therefore 252,000 hours which went into the product in total. Divided by the LOC derived above, we arrive at around 5,5 lines per person per hour.

I found this very interesting, as the outcome is obviously in the same ballpark as the result of the aforementioned article. This also matches my own observation of the team.

So what does this all mean? It means that when talking to people who are not actively programming and are often assuming that the developers should be faster, we have yet another argument on our sides to show that writing even a few lines of code takes up a lot of time.
