+++
title = 'Inference'
date = '2026-02-02T09:40:44-08:00'
draft = false
+++

In the era of *Software 2.0* Linus Torvalds has spoken positively about “vibe coding.”

![Linus Torvalds on vibe coding](linus-vibe-coding.png)

Tools like Claude Code have become default companions for many solopreneurs building and shipping products.

This is a fundamental change in how software is produced.

Developers are now burning thousands of dollars’ worth of tokens to vibe code the next billion-dollar app. As a result, token usage and compute cost have become hot topics of debate across the developer community.

But it’s not clear that any of these numbers reflect what is actually happening under the hood.

To answer these questions, we need to look at inference.

---

## What Is Inference?

When you send a prompt to a language model, the system runs a forward pass through a large neural network.

This process is called inference.

During inference, input tokens are converted into numerical representations and propagated through dozens of transformer layers. Each layer applies attention mechanisms and feedforward networks using learned weights that were fixed during training.

At a high level, inference is repeated forward propagation:

h₀ → h₁ → h₂ → … → hₙ → logits → token

where each step consists primarily of large matrix multiplications and attention operations.

All of the model’s “knowledge” is encoded in these weights. At runtime, nothing is retrieved or searched. The output emerges from numerical computation.

This is where most of the real cost of AI systems lives.

---

Modern inference pipelines are typically divided into two main phases:

- **Prefill**: running forward propagation over the entire prompt and building attention state  
- **Generation**: repeatedly running forward passes to produce new tokens using that state

Prefill is expensive because it processes the full context. Generation is expensive because it must be executed once per output token.

As prompts grow longer and conversations accumulate more history, both phases scale poorly without optimization.

Naïvely serving large language models would require recomputing attention over thousands of tokens for every request, quickly overwhelming available compute.

Inference only becomes economically viable through aggressive optimization.

Caching, batching, memory management, and scheduling are not optional improvements. They are fundamental to making large-scale AI systems practical.

Understanding inference means understanding how forward propagation is structured, reused, and scheduled at scale.

This is where inference engines enter the picture.

---

## Inference at Scale and vLLM

In practice, production AI systems do not run models directly through raw deep learning frameworks. They rely on specialized high-throughput inference systems designed to maximize hardware utilization and minimize wasted computation.

One of the most widely used open-source engines is vLLM.

vLLM is not a model. It is a high-performance runtime for serving language models. Its purpose is to make inference efficient by:

- reusing cached computation  
- batching requests dynamically  
- managing GPU memory intelligently  
- minimizing redundant work

Most importantly for this discussion, vLLM implements aggressive prefix and KV caching. When multiple requests share the same prompt prefix, vLLM can reuse previously computed attention states instead of recomputing them from scratch.

In other words, if two prompts look mostly the same, the second one is much cheaper than the first.

This behavior mirrors how large commercial providers run their systems internally. vLLM makes these optimizations visible and measurable, which makes it a useful tool for understanding real-world inference economics.

---

In the posts that follow, I’ll be starting a series that breaks down the inner workings of vLLM to better understand inference at scale.
