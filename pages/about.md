---
layout: page
title: About
permalink: /about/
weight: 3
---

# **About me**

Hi I am **{{ site.author.name }}** :wave:, a software engineer with over 15 years of experience ranging from Android to back-end to deep learning. My breadth of experience makes me quick to pick up new skills.

I specialize in writing thoroughly-tested, idiomatic code with minimal bugs that is short, organized, and readable, ensuring that anyone else who works on it in the future can maintain a high velocity without getting confused and creating more bugs. But I'm also flexible. For throwaway prototypes, I know how to write fast and messy code to get the job done quick.

With me on the team, your product will be performant, stable, and easy to extend.

<div class="row">
{% include about/skills.html title="Programming languages" source=site.data.programming-languages %}
{% include about/skills.html title="Skills" source=site.data.skills %}
</div>

<div class="row">
{% include about/timeline.html %}
</div>

{% include about/skills.html title="Other skills" source=site.data.other-skills %}