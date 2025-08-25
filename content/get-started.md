---
layout: subpage
title: 'Get Started'
description: 'Direct developers and architects to resources, hands-on guides, and starter templates.'
permalink: /get-started/
hero_links:
  - title: "Why Java for AI?"
    description: "Understand the benefits first"
    url: "/why-java-for-ai/"
    icon: "fas fa-lightbulb"
  - title: "Architectures"
    description: "See proven patterns"
    url: "/architectures/"
    icon: "fas fa-sitemap"
  - title: "Modernization Path"
    description: "Plan your journey"
    url: "/modernization-path/"
    icon: "fas fa-route"
---

# Get Started

Direct developers and architects to resources, hands-on guides, and starter templates.

<div class="subpage-hero">
  <div class="subpage-hero-links">
    {#for element in page.data.hero_links}
      <div class="subpage-hero-link">        
          <i class="hero-link-icon {element.icon}"></i>
        <h3>{element.title}</h3>
        <p><a href="{element.url}">{element.description}</a></p>
      </div>
    {/for}
  </div>
</div>


## Quickstart with LangChain4j and Quarkus

## Tutorial Hub

## Integration Recipes

## Video Walkthroughs