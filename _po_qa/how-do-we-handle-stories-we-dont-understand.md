---
layout: post
title: "How do we handle stories that we don't understand?"
image: /assets/in_thought.png
excerpt: "Product Owners often need to deal with very technical stories that they barely understand. To be successful, they only need to understand a few key aspects."
---
The premise of this question is as follows: As a Product Owner, you are not always writing all the stories yourself. In fact, it's neither uncommon nor a bad idea to have others writing the story for you.
But how do you handle these items, especially if you don't understand them? A classic scenario is one where the developers start writing stories for some technical change they would like to see in the system.

When it comes to this specific scenario, the first question to ask yourself is whether or not you should be reading and prioritizing these stories at all anyways. Typically, Product Owners are domain experts for their product, not for the technology they use. Prioritizing technical maintenance work against product features is extremely difficult even if you do understand all the technical details. I have seen many cases where this is a lose-lose situation for the Product Owner. This does not mean that technical items should not be prioritized at all, but  I think there is a better way to solve this.

## Making room for technical things
Technical items are a bit of an edge case. On the one hand, they are stakeholder requirements same as every other requirement. But on the other hand, these items are usually relevant to all other stakeholders as well, while at the same time being of no active interest to them. A reliable system is something that every stakeholder wants, but it's not something they would prioritize - they would simply assume to get it. So usual prioritization techniques fall apart here.

What to do? I suggest two approaches, which can be used in combination:
- Shifting specific technical tasks to feature work
- Providing a time budget for technical tasks

### Stories need to get bigger to reflect their actual costs
If we deliver something while at the same time accumulating technical debt, we are selling it for less than it is worth. If we ship features with something that we call workarounds, or duct-taping, then we are delaying the costs of fixing that into the future. Perhaps surprisingly, I am not saying that this is _wrong_ per se - as long as it is a conscious decision, I am fine with it. For example, you might want to satisfy a customer by pushing out a quick and dirty fix, with concrete plans to ship a proper fix later on. But things get complicated if that promised land of "the future", where we reduce the debt we accumulate, never comes.

So we need to make stories bigger so that they include at least everything needed to properly deliver it within the scope the Product Owner envisions. 

But this is still not enough: Most products are sold above the price needed to produce it, there is always a margin for profit - and we need the same for our stories. In other words, we need to make our stories even bigger, so that we are not only stopping the upward trend for our technical debt, but have room to actively reduce technical debt. A great way of doing so is the Scout Rule, where a developer leaves a piece of code a tiny bit better than they have found it. Change a poorly named variable, add a unit test or two, split a function if you can - but don't go down the rabbit hole of excessive refactoring. This way, we can guarantee that our code actually improves over time.

It's in the Product Owner's best interest to actively encourage this kind of thinking in the development team.

### A time budget helps with most other issues
The aforementioned strategy already helps a lot, as it puts a stop to producing new technical debt, and starts improving old code as well. But then there are other items which really need doing, but which are not attached to feature development. What to do about these?

I suggest you save yourself the headache of prioritizing these against the rest of your backlog and just hand a portion of the time dedicated to development back to the team. If, for example, that portion would be 20%, then the agreement goes as follows: The team utilizes these 20% on their own, as long as there are no surprises for you. Surprises could be positive, like not using the time during one week and therefore being quicker than expected (that messes with your forecast if it happens infrequently), as well as negative, like trying to "save up" on this extra time to take a full week for the full team "off" for technical things. Surprises like these must be discussed with the Product Owner in advance.

You would then proceed to plan your backlog work as usual, without worrying about the missing time at all - for you, it is as if the team is simply getting slower. You give up any access on that time, but expect the technical debt to go down in turn.

## How to handle things we don't understand
There are scenarios where things are too big to be tackled on the side, be it in a dedicated time budget or as part of normal story work. And there are scenarios where you are confronted with stories that you, yourself, do not understand. How do we deal with this?

This breaks down into what your responsibilities are as a Product Owner.
- You need to be able to make all the information the dev team needs available to them. You do _not_ need to know it all by yourself, but if there are questions, and all you can do is turn around and ask the same questions to another party, then you are a strawman PO and should remove yourself from that equation. For technical issues, this is a no-brainer, since the development team usually knows better than you do what needs to be done. For all other items, this can become dangerous if you lose track about what's important about the feature.
- You need to know why it's valuable. That is the key area that you are concerned with, and where people hold you accountable. If the door slams open, your boss and/or CEO rushes in and asks you why the team is working on that specific thing and not on something else, you need to be able to answer that accurately. You do not need to understand the contents of the issue, but you need to keep asking until you understand the value of it 100%.
- You need to be able to make scope vs time decisions. It's the rule, and not the execption, that things will get messy quickly, and the overall goal that you aimed for will not be reached. In that case, you need to be able to assess whether you reduce the scope of what you are doing, but still deliver _something_, or if you postpone the delivery in favor of keeping the scope intact. If you are unable to make this decision, all your other plans will take a hit.

This is a usual pattern that I often see: The development team convinces the PO that they need to do something, and to make it more palatable, they are being too optimistic with their estimate. And once their feature blows up, the Product Owner has no choice but to postpone everything else, because they are lacking insight and knowledge to make an actual scope decision. Product Owners should tackle this head-on by asking openly and candidly: "When things go wrong - and they will - which parts will I be able to cut in order to keep our plans on track?"


# Summary

- It's OK to have items in the backlog that POs don't understand as long as they understand the value, can make scope vs time decisions and are not pushed into a strawman PO role.
- To make room for technical things, consider budgeting the backlog, shifting tech work to feature work or both


_This post is part of the [Product Owner Q&A](/po_qa/start) series._