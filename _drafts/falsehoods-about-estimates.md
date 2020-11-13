---
layout: post
title: "Falsehoods Tech People Believe About Estimates"
categories: ["Estimates","Write-Up"]
image: /assets/houses.png
excerpt: "In this article, I collect several falsehoods tech people believe about estimates. "
---
There are many lists out there about falsehoods programmers believe - about
[names](https://shinesolutions.com/2018/01/08/falsehoods-programmers-believe-about-names-with-examples/),
[time](https://infiniteundo.com/post/25326999628/falsehoods-programmers-believe-about-time),
[prices](https://gist.github.com/rgs/6509585) - so many, in fact, that there are
[lists about falsehoods
available](https://github.com/kdeldycke/awesome-falsehood). I enjoy reading
them, because some of these falsehoods are fun, some are pitfalls that I ran
into in the past, too - and some are eye-opening.

What I have been missing, though, is a list of falsehoods about estimates.
Estimates are a key part in many organizations - Sales pitches, delivery dates,
staffing, budgeting - there is so much out there that relies on estimates. Yet,
there are a lot of falsehoods out there that people believe - not only software
engineers, but also people who either ask for or work with estimates. This
article attempts to list some of the more important and/or widespread falsehoods
that "tech people" (i.e. people who ask for, provide or work with estimates in
the IT sector) believe.

These falsehoods are listed in no particular order. If you have more, please
[send me a message!](/about/)

# Falsehood #1: A Single Number Is an Estimate.
I have nothing to back this up, but I suspect that JIRA is to blame for this
one: This tool allows only one value for "Estimate", so in many companies,
something like "8 hours, 30 Minutes" passes as an estimate.

However, we do know exactly that whatever this is, it will not take exactly 8
Hours, 30 Minutes and 0 Seconds. But if it doesn't, how long *will* it take?
Will it take between 8 and 9 hours, or more like something between 4½ and 12½
 hours? Without a range to go with it, a single number is not an estimate.

 (This is different for story points, which usually *do* come as a single
 number. However, since those are based on a changing velocity, they represent a
 time range by design.)

# Falsehood #2: Estimates Tell Me When Something Will Be Done.
This is one of the most fundamental falsehoods. If Alice tells Bob that
something is estimated between 4 and 8 hours, it will definitely be done by
tomorrow, right? 

This falsehood is so deeply ingrained in many people, that it seems that a clock
starts ticking as soon as they hear any estimate - as if the work on that item
would start right then and there (which in itself is another falsehood) - and
then they are negatively surprised if they check back on the results later on.

However, Estimates usually only tell you how much work is expected to do
something. Crucially, waiting times are usually either not properly considered
or ignored completely - as they are very difficult to predict in advance. If
Person A does the coding for one hour and Person B does the test for another
hour, the work done is two hours - but the waiting time for Person B to become
available after the coding has been done could be much, much longer.

# Falsehood #3: Customers Favor Small Estimates.
Low Estimates equal low costs. Since a customer wants to pay as little as
needed, they prefer low estimates. This falsehood can easily be debunked by
going to the extremes: Provide an estimate of "One second" for any activity and 
you will quickly discover that the fact that an estimate is small is not the
convincing factor.

The customer is looking to get the highest possible quality for the lowest
possible price. For some customers, the price is more important than the
quality, for other customers it's the other way round. When customers sign a
contract, then it's because they trust getting what they want for the money they
pay - and estimates play a part in building this trust. 

In my experience, customers usually prefer *reliable* estimates over *small*
ones. They are building plans upon your estimates, so if your estimates are
wrong, then their plan collapses.

Consequently, if your estimates are a bit too pessimistic - meaning that you
deliver a  bit earlier than expected - the customer's plan is still valid, there
will simply be a waiting time between delivery and plan execution. But if your
estimates are too small, the customer has to reschedule. (Don't worry, though,
Parkinson's Law makes sure that you usually won't deliver too early!)


# Falsehood #4: If We Keep Failing Our Estimates, We Need to Get Better at Estimating
For me, this is a classic one, as I have been approached countless times by
Product Managers, asking me to train the team to make better estimates, since
they are frequently failing to deliver on their own "promises". 

Whenever I am encountered with a statement like this, I start with challenging
the problem in itself, because the idea of "failing an estimate" sounds an awful
lot like mistaking an Estimate for a Prophecy. Estimates are, by definition,
inaccurate. Beyond that, there are countless reasons why an estimate can be
wrong. Maybe there is scope creep, or unexpected sickness, changes in the team,
instabilities in the product and so on - and, of course, the overarching
question of whether or not we really *need* estimates in the first place.

The best estimation process in the world - if it even existed - can not fix your
chaotic process. Trying to fix the estimate then is like trying to combat the
fever instead of curing the disease it accompanies.


# Adjustment Factors make for more objective estimates.

# Precise estimates are good estimates.

# Estimates include everything.
# Estiamtes only include development activities.



# Falsehood #3: It's Okay to Tweak an Estimate to Make It Fit into the Schedule

# Adding a padding will fix optimistic estimates
# Reducing the numbers will fix pessimistic estimates

# Narrow range equals good estimate

# Estimating is part of the skill set of any software engineer

# Overestimating is worse than underestimating

# The Cone of Uncertainty shows that ranges get smaller the feature progresses.


# Falsehood #3: The Work Needed to Provide an Estimate Is Part of the Development Work
This is a classic response to the question why a company pays for estimates,
even if they are not useful: The work invested in making an estimate is work
saved during development!