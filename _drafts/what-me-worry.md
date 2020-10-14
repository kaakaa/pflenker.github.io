---
layout: post
title: "What, me worry?"
categories: ["Pet Project Sematary"]
image: /assets/what-me-worry.png
---
This first entry in my Pet Project Sematary is already somewhat of an exception: It is not something that I created on my own.
It's a listing that I found in an old issue of the German version of Mad Magazine - which was published in the latter half of the 80s.
I was around 8 or 9 when I have entered the listing, character by character, line by line, hour after hour. 
This was for a Commodore 64, which have gotten very cheap in the early 90s - which is why we got one even though there were more modern
alternatives available. 

Unlike modern computers, the C64 presented you with an environment in which you could actually program something, right after you turned it on. 
So, at some point, every one who had a C64 wrote the following three lines:

```
5 PRINT "HELLO WORLD"
10 GOTO 5
RUN
```

The first two lines define a program which prints out "Hello World" over and over again. The third line actually executes the program.

Each C64 was shipped with a small manual containing a lot of examples on how to do stuff with the C64, and I have typed them all.

## What it was
The aforementioned issue of MAD Magazine printed out the source code of a program which was supposed to output the image of Alfred E. Neuman (after having to wait 20 minutes, no less!). Disks or CDs where not a thing to be shipped with magazines back then, so you had to type it all in on your own.

The actual listing was about 150 lines of code which all looked something like this:
```
500 DATA -27,-11,-23,-6,-28,-13,-22,-6,-20,-5,-12,-5,-27,-14,-26,-13
510 DATA -38,-29,-42,-28,-40,-28,-50,-16,-8,13,0,13,-29,4,-29,9
520 DATA -50,-17,-41,-28,-49,-17,-50,-8,-8,12,0,12,-28,5,-28,13
530 DATA -50,-15,-49,-10,40,-26,42,-17,-4,9,-21,14,5,48,2,44
```

## What I learned
Bugs are a thing. After typing it all in, I finally entered `RUN` and was greeted with:

```
?SYNTAX ERROR IN 80
READY
```
This means that there was some typo in Line 80. I compared the line with the listing, character by character, and I had typed it all correctly - yet it didn't work.
I ended up crying my eyes out on the lap of my mother (did I mention that I was around 8 years old at the time?) who was probably questioning her choice of getting me a C64 then.
It was more than two decades later when I learned that this listing was indeed buggy, and corrected in the follow up issue of MAD, which I never read.

Needless to say, I never saw Alfred E. Neuman on my Commodore.