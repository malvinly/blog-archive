---
layout: post
title: "Levenshtein distance"
date: 2013-12-27 19:38:20 +0000
categories: ["General"]
original_url: https://nivlam.wordpress.com/2013/12/27/levenshtein-distance/
---

Imagine a scenario where a single script is deployed to several hundred different locations. Due to various constraints, this script cannot be centralized, so making a change means I'll need to deploy it to several hundred locations.

But it gets worse. Some of these scripts are customized and include special logic, so I cannot blindly copy the updated script to all locations. In addition to that, most of the existing scripts contain comments such as:

{% raw %}
```bash
#
# This script was created on 1/1/1970 by John Doe.
#
```
{% endraw %}

If these scripts didn't include their own unique comments, I could have compared the file sizes or generated SHA1 hashes for each script to see which were identical and which contained their own special logic. Since each script contains their own unique comments, generating hashes would mean a different hash for each script.

Instead of reviewing each script individually, I can use the [Levenshtein distance](http://en.wikipedia.org/wiki/Levenshtein_distance) to determine how similar the target script is compared to my updated script.

According to Wikipedia, the Levenshtein distance is:

> ... a string metric for measuring the difference between two sequences. Informally, the Levenshtein distance between two words is the minimum number of single-character edits (insertion, deletion, substitution) required to change one word into the other.

Sorting each script by the Levenshtein distance gives me a good indication of which scripts I can safely copy over and which I need to review manually. Overwriting scripts with a Levenshtein distance close to zero gives me reasonable assurance that I won't break anything. While it's not a bullet proof solution, it's better then reviewing hundreds of scripts manually.
