---
title: "Multi-Step Reasoning with Chain-of-Thought Pattern"
description: "An AI reasoning app that breaks down complex problems into intermediate steps before producing a final answer."
complexity: "Intermediate"
use_case: "Knowledge Management"
technologies: ["Quarkus", "LangChain4j", "Ollama", "Tools"]
layout: architecture
---

This demonstrates the Chain-of-Thought (CoT) pattern – the AI is guided (or self-guides) to think through a problem in a structured way, rather than answering in one shot. The Quarkus example implements a structured CoT approach, where the model’s reasoning process is made explicit and possibly each step is handled separately. This pattern can improve correctness for tasks like math word problems, logical reasoning, or troubleshooting, by allowing the AI to tackle sub-tasks one by one.

## Architecture Overview

This pattern combines:
- **Quarkus** for the application runtime
- **LangChain4j** for AI orchestration
- **OpenAI/Ollama** for language model inference

## Key Components

* Quarkus controller/orchestrator: The app might expose an endpoint where a complex query is submitted. The controller orchestrates multiple calls or prompt stages with the LLM.
* LangChain4j managed chain: Instead of one prompt, the logic is split into a sequence of steps (a chain). For example: Step 1 – have the LLM outline a plan or derive facts from the question; Step 2 – maybe use a tool or a calculation if needed; Step 3 – have the LLM produce the final answer based on previous steps. LangChain4j can be used to manage prompt templates for each step or to implement a custom reasoning loop.
* Local LLM: Acts as the reasoning engine. It may be prompted with special instructions like “think step by step” or asked to output a reasoning trace. The model’s intermediate output (the “chain of thought”) can either be purely internal or logged/parsed by the application. In some implementations, the reasoning is kept hidden, while in others the model outputs a structured format (e.g. a JSON with a “thought” and “action” field).
* Tools or Calculators: If the problem requires external knowledge or arithmetic beyond the LLM’s ability, the chain-of-thought approach can include calling out to a tool (similar to the agent pattern). For instance, if step 2 needs a database lookup or a math calculation, the Quarkus app will perform that and feed the result into step 3. The focus might be on purely AI reasoning without external data (using the model’s knowledge), but it sets the stage for more complex agent behavior.
* Controlled output: By structuring the prompts, the developers ensure the final answer is derived from logical steps. This can also allow validation at each step (you could, for example, verify the output of a calculation step before continuing). The result is a more reliable problem-solving AI, as the chain-of-thought method can mitigate the model’s tendency to “jump to conclusions.

<img src="/assets/images/architectures/multi-step-reasoning/overview.png" alt="Architecture Overview" style="max-width: 100%; height: auto; display: block; margin: 0 auto;" />

## Implementation Guide

