---
layout: post
title: ""
categories: [Miscellaneous, Tools]
---
[How a JIRA misconfiguration leaks data of NASA and hundreds of Fortune 500 companies](https://medium.com/@logicbomb_1/one-misconfig-jira-to-leak-them-all-including-nasa-and-hundreds-of-fortune-500-companies-a70957ef03c7). 

> Thousands of companies filters, dashboards and staff data were publically exposed. **It occurs because of the wrong permissions scheme set to filters and dashboards** hence providing their access even to non-logged in users and hence leading to leaking of sensitive data. [...] Some of the companies were from Alexa and Fortune top list including big giants like NASA, Google, Yahoo, etc and government sites as well.

This is not a misconfiguration, though, it's extremely poor UI/UX by JIRA. Creating a filter gives you the option to share it with "Everyone", which sounds like "Everyone in the Company", but means "public". "Everyone in the company" is actually called "Open" - and not even part of the Share Filter UI.