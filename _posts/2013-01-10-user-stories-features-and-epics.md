---
layout: post
title: "User stories, features, and epics"
date: 2013-01-10 08:26:45 +0000
categories: ["General"]
original_url: https://nivlam.wordpress.com/2013/01/10/user-stories-features-and-epics/
---

There have been a lot of discussions in my team recently about the definitions of a user story, feature, and epic. I'm by no means an agile expert, so this is just my opinion and interpretation. I'm the type of person who trusts their intuition, so the way we're currently forced to group our requirements feels extremely awkward to me.

Currently every user story that is created must be grouped under a feature. All features must be grouped under an epic. By the way I've described things so far, it appears there is a hierarchical structure that must always exist: Epic -> Feature -> User story.

To be honest, I hate this. I don't like it at all. If a user story is small enough to implement on its own, why do I need to bother creating a feature or epic? This screams process over pragmatism. If we're forced to always create features and epics, they are no longer features that describe how a system works. Instead, we're using them as categories or buckets of work.

After reading multiple articles around the web, I found [this](http://www.mountaingoatsoftware.com/blog/stories-epics-and-themes) article by Mike Cohn that contains a very simple and concise definition for each.

To me, a user story is a requirement that can stand on its own. It doesn't need to be grouped under a feature or epic. If a user story is too large but not large enough to be considered an epic, it may be broken down into "tasks" that will eventually be absorbed back into the user story after completion. Many people have the opinion that a user story should be sliced into multiple smaller stories instead of tasks. I disagree because that story will be sliced to the point where it doesn't provide any business value on its own. Instead, they become system or technical stories. A user story should be something that delivers business value on its own.

If a user story may be broken down into multiple smaller stories that can provide business value on its own, then it may be considered an epic. I really like the way Mike Cohn described themes, or what we would consider features. They are simply a "rubber band" around a collection related of user stories. There isn't a hierarchy between a feature and epic. Since a feature is simply a collection of related stories, there are times when a feature may be larger than an epic or vice versa.

Neither epics nor features are required. If a requirement is small enough to fit inside a single user story, then that is all there needs to be.

Again, I'm not an agile expert, so this is just my opinion and interpretation.
