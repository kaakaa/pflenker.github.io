---
title: Should you make optimistic estimates?
layout: post
categories: Estimates
---
When being confronted with two alternatives - an estimate which feels optimistic and one which feels pessimistic,prefer the pessimistic approach over optimistic estimates to keep things predictable and avoid chaos. Here's why.

First of all, it should go without saying that you should always strive to make an accurate estimate, based on your current level of information. But then again, *estimation* does not mean *knowning*, and we all know the situation where we have to choose between an optimistic and a pessimistic estimate. For example, you might be using Story Points as an estimate and have to choose between an 8 and a 13 when it really feels like a 10. Or a task could be around 8 hours if everything goes well, or closer to 16 hours of (*insert unlikely event here*) happens.

What do you do? Do you give the optmistic or the pessimistic estimate?

## The case for optimistic estimates
There are a few frequently heard arguments in favor of providing optimistic estimates:
- They build healthy pressure: Optimistic estimates translate into challenging goals, otherwise the sense of urgency would be missing
- They counteract the [Student Syndrome](https://en.wikipedia.org/wiki/Student_syndrome), where the developers procrastinate until the last minute
- Likewise, they prevent [Parkinson's Law](https://en.wikipedia.org/wiki/Parkinson%27s_law), where the work expands so as to fill the time available for its completion.

Additionally, optimistic estimates are usually easier to sell to stakeholders, therefore it's easier to give optimistic estimates.

## The case for pessimistic estimates
Let's take a look at the argument in favor of providing pessimistic estimates:
- They are much more likely to achieve, as there is simply more time available
- They help running the project or product roadmap much more smoothly, as typically other activities are planned after the implementation of a certain feature
- They ensure a better technical foundation, as you are less likely to cut corners  or leave bugs unfixed due to time constraints
- They help preventing late-project dynamics, escalation calls, war room meetings and so on, as it is less likely that the feature will be overdue


## Weighing the arguments
> Motivation (or lack thereof), or procrastination are issues within the development team. Manipulating the estimates will not solve these issues.

It might boil down to management style, but I do not buy into the whole idea of "healthy pressure". If you go out and ask people whether or not they think that there are people who frequently need pressure in order to perform, chances are that they will agree. But if you ask them if _they_ usually need to be motivated by pressure in order to perform, they will most likely disagree. It seems that the consensus is that lazy people who need pressure exist, but they are always "the others".

Apart from the questionable benefit of said pressure, this way of "motivating" employees has no place in an estimate. Estimates are used for plans, roadmaps, sometimes even contracts, and the internal quirks of the development team need to stay out of it. This also applies to the second argument about the Student's Syndrome - it's better to counteract this with more active task management, than to tamper with the estimates.

The arguments in favor of pessimistic estimates mostly boil down to "If the team has more time, they can deliver better work", which is (hopefully) a no-brainer. The arguments against that stem from the assumption that the team will fill up the unused time with unnecessary gold-plating or slacking off, which could both be countered with better task management.

> Overestimated features are still predictable - underestimated features are not

There is also another factor in favor of the pessimistic estimates, which is not self-evident:
Suppose you have estimated Feature A pessimistically, and Feature B optimistically - and both estimates are wrong, so Feature B takes longer than expected, Feature A will be implemented quicker than expected. When will they be done?

In case of Feature A, that is easy to answer: If we assume that Parkinson's Law is in full effect, it still takes as long as it was originally estimated. In other words, there is an *upper bound* for how long Feature A will take. There is also a lower bound for how long the feature will take. That bound is certainly not zero, since *some* work needs to be done, but otherwise we don't know (if we knew, we wouldn't be estiming. We should quit and become professional lottery players).

But what about Feature B? All we know about its lower bound is that it is bigger than the estimate. But there is no upper bound as to how long the feature will take - especially if you take destructive dynamics into account where the developer needs to give frequent updates, sit in on escalation meetings and so on. Additionally, once the plan for Feature B falls appart, Features C, D , E and all others queued in after Feature A are also delayed. Not only that - all attached activities such as QA, UAT and more are delayed as well.

From a cost and risk perspective, it is much easier to plan around Feature A than Feature B.

## Conclusion

> Even though optimistic estimates make you look good, always go for pessimistic estimates when in doubt. That will make you look even better.

When in doubt, be pessimistic about your estimates. Optimistic estimates give your stakeholders a warm and fuzzy feeling, pessimistic estimates give then a more reliable building block to assess their project risks. Over the long run, it will make you look better, too, as people will gain confidence in your estimates.

If that still does not convince you, remember that the software industry as a whole has an underestimation problem - whenever we think we give an accurate and not overly optimistic estimate, it is very likely to be an underestimation.

In my teams, I encourage pessimistic estimates with the following rule: When you have two clear alternatives in mind, go for the pessimistic one. In case we are estimating with story points, the rule is: If the estimates are off by one level of complexity we go with the higher one without discussion. If the spread is bigger the team still needs to discuss.

