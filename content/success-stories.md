---
layout: subpage
title: 'Success Stories'
description: 'Real-world enterprise AI integration using Quarkus.'
permalink: /success-stories/
hero_links:
  - title: "Why Java for AI?"
    description: "Understand the benefits"
    url: "/why-java-for-ai/"
    icon: "fas fa-lightbulb"
  - title: "Architectures"
    description: "Implementation patterns"
    url: "/architectures/"
    icon: "fas fa-sitemap"
  - title: "Get Started"
    description: "Begin your own journey"
    url: "/get-started/"
    icon: "fas fa-rocket"
---

# Success Stories

Real-world enterprise AI integration using Quarkus.

{#for story in site.collections.success-stories}
<div class='success-story-card'>
  <h3><a href='{story.url}'>{story.title}</a></h3>
  <p>{story.description}</p>
</div>
{/for}

