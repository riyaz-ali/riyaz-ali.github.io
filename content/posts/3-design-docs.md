---
title: Design Docs
date: 2019-01-22
author: riyaz-ali
background: https://source.unsplash.com/aJTiW00qqtI
description: |
    In this article, I've tried to explain what a design document is
    and how you could prepare one!
aliases:
- /design-docs.html
tags:
- SoftwareEngineering
- Documentation
keywords:
- SoftwareEngineering
- Documentation
- Design Documents
---

A design doc — also known as a technical spec — is a description of how you plan to solve a problem. It's one of the most useful tool for making sure the right work gets done.

## It's just more overhead man..

🤷‍♂️ people often claim that it’s because they’re saving time by skipping the spec-writing phase.

But what they don't realise is that in the long run (even for a smaller project that's true), they might be spending a lot more time fixing, iterating and communicating all the same stuff (over and over again) what they could have done once if they had written a proper spec for it.

Joel Spolsky ([@spolsky](https://twitter.com/spolsky)) has written [quite extensively] about this in his [article](https://www.joelonsoftware.com/2000/10/02/painless-functional-specifications-part-1-why-bother).

## So, why does it matter?

A well-structured design doc can save you days, or even weeks of _wasted time_!

The main goal of a design doc is to make you more effective by **forcing you to think** through the design and gather [early] feedback from others.

Point is that when you design your product in a human language, it only takes a few minutes to try thinking about several possibilities, revising, and improving your design whereas when you _design_ the same product in a computer language, it takes _weeks_ to do iterative designs / changes.

Another big reason of having well-written design docs is that they _save us time communicating_ the stuff by making sure that **everyone is on the same page**, and that can help us build great products and innovate quickly. Everybody on the team can _"just read the spec"_.

What happens when you don’t have a spec is that all this communication _still happens_, because _it has to_, but it happens ad-hoc..

## How do I go about writing a design doc?

Umm... I won't go into _too much_ detail. There are already [several](https://www.freecodecamp.org/news/66fcf019569c) [good](https://blog.nuclino.com/how-to-write-a-technical-specification-or-software-design-document-sdd) [articles](https://people.eecs.berkeley.edu/~kubitron/courses/cs162-F06/design.html) on the internet that does that. I'll try and summarize the points (and maybe add a few from my side) into the following list -

- Try and stick with _high-level_ overview without diving too much into implementation detail
- Define what service you're building and what are the objectives
- Provide a context to the readers, describing where the service fits in the overall architecture
- Outline goals and non-goals of the service and make them quite explicit
- Give a high level technical overview. Use user / case stories to depict scenarios
- Highlight your dependencies (what other services your service will depend on) and tell "why"
- Describe your choice of stack (but don't dive too much into technicalities) and the reason why you chose them
- Highlight any potential issues / open questions and get feedback from peers
- Add links to other sites, references or articles that might be helpful
- Get feedback early-on rather (and perhaps on each step)

I personally found that following ☝️ steps is generally very helpful and makes one more productive than doing it the other way

### References

- [How to write a good software design doc](https://www.freecodecamp.org/news/how-to-write-a-good-software-design-document-66fcf019569c/) by [Angela Zhang](https://twitter.com/zhangelaz)
- [Painless Functional Specification - Part 1](https://www.joelonsoftware.com/2000/10/02/painless-functional-specifications-part-1-why-bother) by [Joel Spolsky](https://twitter.com/spolsky)
