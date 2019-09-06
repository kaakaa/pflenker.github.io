---
title: Why you shouldn't use AI for automated content filtering
layout: post
categories: [Ethics, "Computer Science", Censorship, AI]
---

One of the big questions of our time can be summed up into one sentence: Should you block or filter illegal content which is uploaded to the internet? 

The pro argument is obvious: If it's not there, it can't be distributed. If it can't be distributed, there is less demand, and therefore less supply. The drastic pro argument that is always brought to the table is: If there is less demand for child pornography, children will be protected. 

The counterarguments are also well known by now, some of them are: The same effect can be achieved with less drastic means, and it's basically a censorship platform waiting to be misused. In the times of Trump, Johnson and Bolsonaro, the argument that it will definitely not be misused by future governments, pinky promise, seems less and less compelling. 

I don't want to argue for any side here. Instead, I want to comment on the proposal brought forth by [Wolfgang Kreißig, Chairman of the Commision of protecting youth in media (KJM)](https://www.heise.de/newsticker/meldung/Soziale-Medien-Kontrolleure-draengen-auf-Einsatz-von-KI-zum-Jugendschutz-4514660.html) (even though I find it telling in the light of the above counter argument, that within the article, copyright infringement is put on the same level as child porn and revenge porn). 

The idea is basically the following: The KJM has discovered that there are AIs which could be used for recognizing unwanted material. 

> He has been surprised by the "ad-hoc high recognition rates", commented [a technician who has researched these AIs]. These had reached 83-90% for pornography, 77% for violence and 78% for extremism.

The underlying question in this article is: If the technolgy is there, why aren't we using it? 

### Why high recognition rates still lead to overblocking
The answer to that lies in [Bayes' theorem](https://en.wikipedia.org/wiki/Bayes%27_theorem), a handy little rule which is unfortunately not always taught at school. Whenever multiple probabilities at play, Bayes is not far.

Let's put Bayes to use. First, we are going to define the probabilities at play here/

Suppose 0.01% of all posts being uploaded contain violence. You have an AI that has a recognition rate of 77%, that means: of 100 violence-containing posts, 77 are correctly identified as containing violence, and 23% are identified as harmless ("true negative"). For any harmless post, the AI identifies 99% correctly as harmless, and 1% as containing violence ("false positive"). For any post being uploaded, the AI tells you that it contains violence. How likely is it that this is true?

Note that I have made up two numbers here: The 0.01% of posts containing violence, and the rate of fales positives. In order to fully judge the capabilities of an AI - or any test, for that matter - these numbers need to be known. Knowing only the recognition rate doesn't tell you enough, as you will see in a second.

### Knowing only the recognition rate of a test is not enough

I won't bore you with actually applying Bayes' Theorem here, this is not a Math blog after all. You can enter the numbers in any Bayes calculator out there, or ask someone with a math background near you to do it for you.

If you do, you'll get the following results: The probability that a post is harmless when it is identified as violent is 99,2%, and the probability that a post is violent if it has been classified as violent is around 0.8%. In other words: If your AI blocks 1000 posts, then 992 of them are most likely actually harmless, and only 8 contain real violence.

This puts the "impressiveness" of the recognition rates into perspective. In medicine, a recognition rate of 77% would not be impressive, it would be abysmal. 

As anyone living in Twatt, Scotland or Scunthorpe, England, can tell you, overblocking can be a serious problem. When AI comes into play, however, things get more complicated as it is usually not possible to actually find out why the AI has blocked your content: If you're blocked, you're blocked.

Any AI in charge of recognizing and filtering bad content will overblock, and it won't even tell us why.