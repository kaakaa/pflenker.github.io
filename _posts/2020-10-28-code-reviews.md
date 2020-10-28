---
layout: post 
title: "Code Reviews Should Be the Exception, Not the Rule"
categories: ["Quality", "Engineering" ] 
image: /assets/code_review.png 
excerpt: "Code Review is an important part of every software development process.
However, it catches mistakes and errors too late and its place as a default step
during development should be replaced with other engineering practices."
---
Code Review is an engineering practice that is the de-facto part of most
engineering  processes world wide. The idea is simple: Two pairs of eyes are
more likely to catch mistakes, code smells and conceptual errors than one.

When forming a new team, I would definitely start by making Code Review a
mandatory step during development, but I would definitely not *end* there. As
soon as the team is up and running and has a good development process in place,
Code Review should become the exception, and not the rule.

As we will see in this article, Code Review solves various problems which can
also be solved by a combination of automation, pairing and exceptional code
reviews.

# How Much Fixing a Bug Costs Depends on When It Is Discovered

How much it costs - both for the company and for the customer - to fix a bug
depends more than anything else on when it's discovered. On the one end of the
spectrum, you discover the bug while you are thinking about it. Let's say you
want set the size of an object based on the user's input. You might think along
the lines of: *"OK, `width` and `height` hold the dimensions the user provided,
so I am going to apply these values to the object.  No, hold on, these
dimensions are `strings`, I need to validate and convert them before using,
which also means I need some more error handling..."*

By the time you have taken to read this sentence, this inner monologue would
already be over. Time cost: A fraction of a second.

On the other end of the spectrum, there is an error in production. Let's assume
you did the validation of the dimensions wrong and introduced a subtle bug. 

> In case you are wondering how something as simple as input parameter
> validation could create a subtle bug: If you use JavaScript's `parseInt`
> function without a second argument, it will produce wrong results for very
> specific numbers. More specifically, whenever the user provides "09" or "08"
> as an input, it will be converted to 0, and not 8 or 9, respectively. Check
> [here](https://stackoverflow.com/questions/850341/how-do-i-work-around-javascripts-parseint-octal-behavior)
> for more infos about the why.  
>
> This is a great example, because this is one of the things that you simply
> have to *know* in order to avoid it, it's not a bug that you can discover by
> yourself without having the knowledge.

It takes a lot of time and effort for the customer to be able to find out that
they are dealing with an actual, reproducible bug in your software. Then the
customer has to file a bug, which then goes to your company's support team and
then to the dev team - which is at that time already working on a completely
different feature, meaning that someone needs to context switch in order to
reproduce, find and eventually fix the bug.

And if you are working in an environment where it's not common for a user to
report a bug - for instance on a regular web project - you might be losing
frustrated customers without even noticing.

# Most Engineering Practices Directly or Indirectly aim to Reduce the Costs of Fixing a Bug
When it comes to engineering practices and processes, we almost always strive to
reduce the costs of fixing a bug. We're showing our work to customers or their
representatives because it's cheaper to fix any misunderstandings we have, or
any bugs that occur when they click where no one has clicked before, right then
and there rather than in production. We test our stuff, because it's cheaper to
fix bugs there before we start bothering customers. And we do Code Review,
because we believe that we will catch things there which have been overlooked
during development, but which will cost much more if found later down the road.

Of course, fixing bugs is not the only reason we do Code Review - we want to
ensure that the produced code is in line with the rest of the code in the same
project, that it conforms to certain patterns, doesn't contain any code smell,
contains enough tests and so on. Some of these do not contribute to the goal of
reducing bugs - the idea in these cases is to keep the code maintainable and
extendable enough for the future.

# Arguments Against Code Review
There are several arguments against Code Review. Many teams rarely acknowledge
these issues as impediments for their speed, but rather see them as a necessary
drawback that you just have to live with.

## Code Review is Too Late
Code Review is great for finding these issues, but it comes too late. With the
exception of the tiniest of findings, like a single typo, every issue found in
development means going back through your code, modifying multiple places,
checking if the tests still cover every case and so on. Rather than just hitting
backspace a few times when you made a mistake, you now have to fix all the found
issues at the same time, while being very careful not to introduce new issues.

## Code Review Introduces Waiting Times
One often-overlooked thing when producing software is the time any issue spends
waiting between two steps. A 15-minute-change which is approved within 5 minutes
of Code Review takes 4 hours and 20 minutes, if it takes half a day for the
person responsible for the code review to have time. Things get even worse if
the reviewer requests changes which they will then review a second time -
because unless you are twiddling your thumbs until the review is done, you won't
be available the minute your code has been rejected, which means even more
waiting time for the feature.

There are strategies to minimize these waiting times, but nothing beats *not
having any waiting time at all*.

## Code Review Introduces Context Switching
Strongly related to the aforementioned waiting times is the issue with context
switching. Anyone reviewing your code needs to switch from whatever they are
currently working on to your code and understand what problem your code is
trying to solve. While they are doing this, you are already context switching to
another issue yourself - figuring out what is being asked from you there. Once
the code review is done, things happen in reverse - the reviewer switches back
to their code, you switch back to yours. This goes back and forth until all
issues are eliminated.

# Automate as Much as Possible
Instead of relying exclusively on Code Review to find code style mismatches,
code smells and so on, you should start automating as much as possible. Your
development environment should warn you immediately if you violate any of the
rules that have been set up for your repository, or if you are using known anti
patterns or gotchas for the language you are using.

You can and should configure your repository in a way that makes it impossible
to accidentally push code that does not conform to the code guidelines or does
not contain enough tests. Conforming to the rules should be easy and quick,
breaking them should be painful and cumbersome.

This reduces the time between you produce the bug in your head to discovering it
to some time between you have written it or when you try to push your changes.

# Pair for Instant Code Review and Knowledge Sharing
Pair Programming is the one technique you should consider instead of a dedicated
Code Review step. Pairing forces you and your partner to collaborate on an issue
- which means that conceptual errors and misunderstandings are much more likely
to surface, since you have to explicitly talk about what you try to achieve with
the code you are writing. If there are certain quality standards that are
required for the code, but for some reason impossible to automate, then pairing
helps to spread that knowledge, because team members are reminding each other
about it.

From the perspective of speed, pairing is the better choice even when we are
being pessimistic about the costs: Even if the amount of work invested in the
feature would double, because two people are now involved, the actual time in
which the feature mentioned earlier is completed would go down from 4 hours and
20 minutes to, still, 15 minutes. Remember that customers do not care about how
busy your team is, but how quickly they can ship high-quality features, so we
should optimize for that over optimizing for coding time.

# When to do Code Review
So, when *should* we use Code Review? I am not advocating against Code Review in
general, I merely would like to see it gone as a default part of the development
flow. Here are three cases where Code Review still makes sense.

## When the Knowledge is Distributed Unevenly
If there are very few persons in the team who can effectively judge the quality
of the code for any given repository, a dedicated Code Review step makes sense.
Otherwise, you would make these few people a bottleneck for development, since
they always have to be part of every pair that produces code. Moving this
bottleneck to the Code Review step is, albeit temporarily, a good solution.
However, your team should actively work on spreading the knowledge, so that the
number of experts on the code quality increases until the bottleneck is gone.

## For Exceptional Cases
Sometimes, the problem, the solution or both is too complex even for a pair to
handle successfully, so they would like to have an additional set of eyes on
their code after it's done.  In these cases, Code Reviews make a lot of sense
and should be used generously.

However, as usual with "exceptional cases", you have to look out for the risk of
the exception becoming the norm, which is usually a sign of hidden knowledge
gaps within the team.

## When Pairing is Impossible
One main pillar of my stance outlined in this article is Pair Programming. If
that is not possible, a dedicated Code Review step is the next best alternative.
For example, the maintainer of an Open Source project on github can not pair
with any contributor to that repository, so they need to rely on Code Review to
ensure the incoming changes fit the rest of the code.




