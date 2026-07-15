---
layout: post
title: "Using \"agile\" as an excuse to write poor code"
date: 2012-09-12 07:10:54 +0000
categories: ["General"]
original_url: https://nivlam.wordpress.com/2012/09/12/using-agile-as-an-excuse-to-write-poor-code/
---

Every time I hear the phrase "we're using agile", I cringe. Agile is not a noun. I was reviewing the slides for a presentation titled ["Agile or Fragile"](http://hp.usersummit.org/nashville/events/20110831/) and a few points stood out:

**You might be fragile if...**

- Schedule takes precedence over quality in a "whatever it takes to make the date" sprint.
- You attempt to run a virtual agile team that is spread across a large enterprise or geography.

**Add value to customer**

- The focus is on delivering code to production in two-week sprints in order to meet project timelines.
- The focus is on delivery period.
- Customer value is limited as they must deal with buggy software until repaired.

**Success requires participation**

- Issues are pushed to the backlog as nothing can get in the way of delivery.

**Collaboration not just co-location**

- Not all team members are located together.
- Teams often work together in absolute silos with little accountability for the quality of their delivery.

**Quality matters**

- Progress is measured by the completion of a sprint on time.
- Schedule over quality becomes the unspoken imperative.
- Ship and repair is the deployment strategy.

Unfortunately, I've had the opportunity to see the fragile points mentioned above happen on more than one occasion. To some people, being agile gives them a chance to write poor code and add bugs or technical debt to the backlog. These poor development practices are not unique to agile methodologies, but since the term "agile" has a loose definition, it gives developers a chance to proclaim anything as agile.

I do not doubt the success of agile methodologies if done correctly. I am completely in favor of the principles behind the Agile Manifesto. However, it should be the developers responsibility to be able to identify when they're incurring technical debt faster than they can address it. You are accountable for what you deploy to production. Do not use the "continuous delivery" principle as an excuse to deploy poor code.
