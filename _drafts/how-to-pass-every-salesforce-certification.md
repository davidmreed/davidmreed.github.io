---
layout: post
title: How to Pass Every Salesforce Certification; or, How are Exams like Code Coverage?
---

The title of this post is meant to be a bit inflammatory.

If you're looking for the "too long; didn't read" summary, here it is: pass the exam by studying, understanding, and implementing the material on the study guide; and by understanding the exam format. That's it.

With the quick summary out of the way, I'm going to ask you to indulge me in a roundabout metaphor. Here's where I start: why does Salesforce make use hit 75% code coverage before we can perform a deployment?

Code coverage acts as a *proxy measurement*. It's a metric on its own of limited utility, and if the unit tests generating it are lousy, it's worse than useless - it's outright deceptive. But it has value because it measures that we've done specific things in accordance with expectations. We've built our unit tests, they pass, and they're sufficiently clever to cause a supermajority of our code to execute in test context. And it's easy to measure.