---
layout: post
title: Questions to ask before providing an Estimate
categories: [Estimates]
---
Estimates are a frequent source of discussion in the industry. This article helps you identifying core issues people have when it comes to estimates, and how to address them.

### Do you need an Estimate or a Plan?
>Ask if they actually need an estimate, or if they need a plan to meet a certain goal. This helps you to give a more meaningful and useful response.

MANAGER: "How long do you think this Feature will take?"    
ENGINEER: "With the current scope, it will be done by August."  
MANAGER: "What? That is unacceptable! We need it for the trade fair in July!"  

This exchange probably sounds familiar, and I have seen and heard it multiple times in many different variations. The bottom line here is that the people involved have no clear concept of a *plan* and an *estimate*.

An *Estimate* is an educated guess how long something will take based on the current level of information. There are different techniques to arriving at a good and reliable estimate, but a crucial point is that the estimate is not influenced by external factors. Don't kid yourself, the Software will not write itself, and it will not go any faster just because someone really, really wants it to. Hoping that things will be better is a terrible strategy! Subjectivity, optimism and hope needs to stay out of estimates as much as possible (In fact, you should [never be optimistic when giving an estimate]({{ site.baseurl }}{% post_url 2019-07-29-should-you-make-optimistic-estimates %})!)

A *Plan*, however, typically includes the subjectivity which needs to be omitted from the Estimates. Estimates are the foundation of a good plan, and a plan can be formed by questioning the estimates. What the manager from the excahnge above was really asking about was a plan to come up with something that can be shown at the trade fair - said plan should have the engineer's estimate as a basis and could include reducing the scope, increasing the team and so on. To come up with a good plan, you need a reliable estimate.



### Do you need the Duration or the Delivery Date?
> Ask if they need to know when to get it, then make sure to mention contributing factors, such as handover, in your response.

The difference between how long something takes, the *Duration*, and when to get it, the *Delivery Date* is a cause of many arguments. So you have your estimate in Story Points, hours, ideal days, man-months, you name it - but what does it tell you? For example, if your estimate is 36 hours, and you start working at noon, can I expect it next day at midnight? Probably not - people need to sleep at some point. A less obvious example is a feature where the implementation is easy, but testing relies on a specialist from another team who is hard to reach.

Estimates tend to not include factors such as handover, vacation, sickness and so on. Both things are important: The Duration usually translates directly into costs, but the Delivery Date usually has a direct impact on customers or stakeholders.

Managers requesting an estimate are typically talking about the latter, but engineers are focussing on the former. The result is that managers are planning their projects or releases based on wrong assumptions, and as soon as they fall apart, the fingerpointing begins.

### What do you need it for?
> Ask what they need the estimate for, as this sets the stage for all your estimating efforts.

This is probably the single most important question to ask. Managers often don't realize that a great deal of the estimation process hinges on this very question. The range goes from "I need a rule of thumb estimate to gauge whether or not my feature idea is 'worth it'" to "I need to write how much hours we are going to spend in this legally binding contract".

The purpose of an estimate has a direct influence over the required precision and accuracy of the estimate, and it tells you a lot about the expectations of the person asking for the estimate.