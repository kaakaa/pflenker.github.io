---
layout: post
title: "Grow Your Team With Staff Liquidity"
categories: ["Team Building", "Leadership", "Write-Up"]
image: /assets/data_points.png
excerpt: "Staff Liquidity provides an alternative or addition to adding more people to a team, by focussing on raising the level of experts for certain roles within the team."
---
It's a typical complaint from any Engineering Team: We are too small, we need
more people to deliver high quality software faster. From the team leadership
perspective, things get complicated fast: What kind of roles do we need to fill?
(Don't forget to [take into account that there is more to consider than just the
job title!]({% post_url 2020-10-21-focus-on-the-right-roles %})) How high is the
risk that this role will eventually run out of work, since there is a bottleneck
elsewhere in our team, or because customer demand changes? And perhaps most
critically, does the new person arrive on time? Adding new people to the team
means, even if we lived in a world where a recruitment process would
nonexistent, that we have to invest time of other team members to ramp up the
new hire. For the first few months, we will be investing more time and effort
than we will be getting out of it.

Staff Liquidity can help address these issues, by approaching the problem from
the perspective of knowledge sharing and dependency reduction within a team. The
excellent book
["Commitment"](https://www.amazon.com/Commitment-Novel-about-Managing-Project/dp/9462410038/ref=sr_1_1?dchild=1&keywords=commitment+novel&qid=1604002115&sr=8-1)
explains the basics of this in great detail, and in this post, I want to
highlight how to use this in an engineering team.

Staff Liquidity requires the following three steps:
- Assessing the risk
- Identifying and prioritizing improvement areas
- Making room for knowledge transfer

# Assessing the risk
The first step to put Staff Liquidity into action is to find out what you are up
against. Make a list of all the different steps in your development workflow,
from start to finish - for example:

- Requirements Analysis
- Development
- Testing
- Deployment

Of course, you can go into much further detail if needed. You could, for
instance, ask about different facets of testing, such as TDD, exploratory
testing and test automation, or about different aspects of development, such as
different languages and frameworks used by the team.

Then ask every team member to rate their own expertise in each area. Don't worry
too much about people being too optimistic or unrealistic about themselves -
things will even out on average. Make sure that you provide the team with enough
options to choose from - a scale of 1-3, for example, would lead many people to
assess themselves as "2", because even if they think they are good, they might
have doubts if they are good enough for the highest possible grade.  
The easiest way is to ask the team to rate themselves on a scale of 1-10, where
1 means "I have no idea how to do this", and 10 means "This holds no secrets to
me".

## Where Do You Want to Grow Today?
The second question to ask is, for each area, whether or not that person is
interested in diving deeper to learn more in that area. It's a difference
between being able and willing to learn anything and everything, and being
actually interested in and intrigued by a specific area, so the answers to this
question will help you a lot with the next step.

# Identifying and Prioritizing Improvement Areas
Whatever scale you use, when evaluating the results, you should treat someone
who has given themselves a ranking from the top third of the spectrum an Expert.
For a scale of 1-10, this means that everyone who rated themselves 7 or higher
would be an expert.

Then you simply count the number of experts in each area, and you classify the
risk at follows:
- Any area with three or more experts is green. There are enough experts around.
- Any area with one or two experts is yellow. The risk that one expert goes on
  vacation and the other one is sick is too high and needs to be addressed.
- Any area with no expert is red. Obviously, not having an expert in any
  critical step of the process is a tremendous risk, that should be mitigated as
  soon as possible.

We will combine this with the results of the second question, to ask ourselves:
Do we have enough people who would be willing to become experts in these areas?

This gives us the following matrix, where an "Expert" is someone with a high
enough self assessment, and a "Learner" is someone who is not an expert, but
interested in growing and diving deeper in that area. We say that we *don't have
enough learners* if not enough people are interested in learning to generate
enough experts.

|                           | **Enough Learners**   | **Not Enough Learners** 
| **Enough experts**        |   Green Zone          | Green Zone          
| **Not Enough Experts**    |   Yellow Zone         | Red Zone
| **No Experts**            |   Orange Zone         | Red Zone 


## The Green Zone: Nothing to do Here!
Areas in which we have at least three experts are considered the Green Zone. The
risk for bottlenecks is low, so is the so-called truck factor, which indicates
how messed up your project is, should a key person be run over by a truck (You
can also call it the Lottery Factor, in which case it would indicate the risk if
a key person wins the lottery and leaves the company).

## Yellow and Orange Zones: Build Up Those Experts!
The Yellow Zone has less than three experts, the Orange Zone has no expert at
all. On the other hand, there are enough people interested in becoming experts
to fill those gaps. All we have to do is make room for learning - depending on
the zone, people will either learn from existing experts, or need to get some
external training or learning to become an expert in the area. We will look into
how to make room for learning below.

Generally speaking, the Orange Zone should be tackled first, since having no
experts for a certain area is generally worse than having not enough experts.

## Red Zone: Houston, We Have a Problem
The highest risk for any team is a red zone, where experts are needed, but not
enough people are willing or able to fill in this role. For all practical
reasons it doesn't matter whether or not we already have one or two experts for
the area - the stress that lands on their shoulders by being the bottleneck all
the time will soon drive them away.

It's important to fill these gaps as quickly as possible, and if really no one
in the team volunteers to step up and fill the gap, there is no other choice but
to look for help from outside of the team.

# How to Make it Happen
The previous steps of this assessment has resulted in a prioritized list of
areas that need improvement. Now, how do we best build up experts in the area
that we need?

Teams usually make the mistake of letting Seniors do the stuff they are experts
in. After all, they are the ones who are most efficient in solving the issue!
While this might be true, this also prevents the learning and knowledge sharing
that needs to happen.

So instead, by default, every aspiring expert should try to get their hands on
an issue of their future area of expertise. They should work with it as best as
they can, and rely on feedback from existing experts to learn.

In the beginning, Pair Programming between experts and learners helps
establishing a baseline of knowledge within the learner. Once both the learner
and the expert are confident enough, the learner can start working on their own
(depending, of course, on the team's way of working), and frequently pull in the
expert for opinions.

Another piece of advise is to see features as learning opportunities instead of
some work that needs to be ticket off some To Do List. That way, items which are
already done and shipped can still be passed to the expert for review. This
doesn't block the release of said feature, and it's obviously not the goal to
introduce late changes to the code, but it creates an opportunity for the expert
to share more knowledge with the learner.