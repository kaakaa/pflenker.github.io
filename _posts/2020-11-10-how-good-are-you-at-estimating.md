---
layout: post
title: "How Good Are You at Estimating?"
categories: ["Estimates", "Write-Up"]
image: /assets/questions.png
excerpt: "People who frequently ask for estimates often don't grasp how difficult it is providing one. This test helps you to find out how good of an estimator you are."
---
The following quiz will help you in finding out how good of an estimator you are.
Let's say a good estimator is confident that their estimates are 90% correct,
in other words: 9 out of 10 estimates are correct.

For the following list of 10 items, please provide a range - a low and a
high estimate - that gives you a 90% chance of including the correct answer.
Please be careful to not make your ranges too wide or too narrow.

Please do not cheat - after all, we are assessing your estimation skills, and
not your googling skills. This exercise is timeboxed to 10 minutes, after
which you should be done.

#  The Items
- Surface Temperature of the Sun
- Latitude of Shanghai
- Area of the Asian continent
- The year of Alexander the Great's birth
- Total value of Euro banknotes in circulation around the Eurozone in May 2019
- Total volume of the North Sea
- Worldwide box office receipts for the movie *Avengers: Endgame*
- Total length of the coastline of the Pacific Ocean
- Number of book titles published worldwide every year
- Heaviest blue whale ever recorded

# Assessing the results
You can find the answers to this quiz at the end of this article. Please award 
yourself one point for every estimate where the actual value was within the range
you have provided.

An estimator with 90% confidence should have 8 or more answers correct. (The math
behind this is a bit complicated - an estimator who is right 90% of the time has
a 93% chance of getting 8 or more answers correct).

Don't feel bad if you failed horribly at this quiz - most people do. In fact, 
I have taken this quiz multiple times, and even knowing the pitfalls doesn't make
you immune.

Here is a distribution of answers from 35 people who took this quiz during one
of my trainings:

![image](/assets/2020-11-10-estimates.png)

As you can see, there is no one who is really a 90%-Estimator. Most people get
either nothing or one thing right. This matches the observations Steve McConnell
describes in his excellent book "Software Estimation", where the original quiz
is coming from.

# Takeaway #1: Our Ranges Are Too Narrow
The first take away from this exercise is that we are making our ranges too
narrow. We believe that an estimate with a narrow range is a better estimate
than one with a wide range - but the opposite is true: The range of an estimate
represents our level of uncertainty. 

Note that the description said nothing about making the ranges narrow - you were
simply asked to make the range not too wide or too narrow - just enough so that 
there is a 90% chance of the actual value being within that range.

# Takeaway #2: We Are Overconfident
The second take away is that whenever someone is 90% confident about their
estimates, chances are very good that they are instead much worse at estimating.

In reality, that means that whenever someone is "super confident" about an
estimate, it doesn't really mean anything at all! The dangerous thing is that we
actually build plans on top of these estimates based on the fact that whoever 
made the estimate said that they were super confident when creating them.

# Takeaway #3: Estimating Stuff is *Hard*
The third take away is that estimating stuff is hard. But how does estimating
the surface temperature of the sun relate to estimating software projects?

It's not uncommon for people to ask for an estimate at a time where the
estimator has to make countless assumptions to be able to provide an estimate.
For example, the question of who will actually be doing the work in question is
one of the top influences on estimates. This means that an estimator frequently
has to make assumptions about who will be doing it - sometimes, this means
making assumptions about people who aren't even onboarded yet! 

Estimating the surface temperature of the sun, by contrast, doesn't sound too
difficult now...

# The answers

- Surface Temperature of the Sun: [Around 5,778 K, 5,505 °C, 9,941 °F](https://en.wikipedia.org/wiki/Sun)
- Latitude of Shanghai: [31°13′43″N](https://en.wikipedia.org/wiki/Shanghai)
- Area of the Asian continent: [44,579,000 km², 17,212,000 sq mi](https://en.wikipedia.org/wiki/Asia)
- The year of Alexander the Great's birth: [356 BC](https://en.wikipedia.org/wiki/Alexander_the_Great)
- Total value of Euro banknotes in circulation around the Eurozone in May 2019: [€1.231 trillion](https://en.wikipedia.org/wiki/Euro_banknotes)
- Total volume of the North Sea: [ 54,000 km³, 13,000 cu mi](https://en.wikipedia.org/wiki/North_Sea)
- Worldwide box office receipts for the movie *Avengers: Endgame*: [$2.79 billion](https://theconversation.com/avengers-endgame-is-nowhere-near-the-worldwide-box-office-record-heres-why-120749)
- Total length of the coastline of the Pacific Ocean: [135,663 km](https://www.cia.gov/library/publications/the-world-factbook/fields/282.html)
- Number of book titles published worldwide every year: [Approximately 2,200,000](https://ipfs.io/ipfs/QmXoypizjW3WknFiJnKLwHCnL72vedxjQkDDP1mXWo6uco/wiki/Books_published_per_country_per_year.html)
- Heaviest blue whale ever recorded: [190,000 kg](https://uk.whales.org/whales-dolphins/record-breakers/)

<small>The quiz is a slightly revised variant of a quiz from *Software Estimation*
by Steve McConnell (Microsoft Press, 2006), which is &#169; 2006 Steve
McConnell. All rights reserved.</small>