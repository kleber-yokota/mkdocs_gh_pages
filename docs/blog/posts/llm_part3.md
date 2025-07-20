---
draft: false
date: 2025-07-19
categories:
  - Datastack
  - LLM
tags:
  - LLM
  - ollama
  - langgraph
  - homelab
  - datastack
---
# Performance Test Generating 1,000 Rows with Local LLM (5070Ti)

I ran a performance test to generate 1,000 rows of data using a local LLM on a 5070Ti. Since generating everything in one go often results in incomplete outputs, I limited the generation to 20 rows per call.

## Two setups were tested:

1. A single node running in a loop  
2. Dynamically creating the graph on each call

Because everything is running locally, the loop version was clearly faster — around **25 seconds total**, compared to **45 seconds** when rebuilding the graph in parallel. That said, if you're running things through an API or have multiple GPUs, the dynamic/parallel strategy might actually make more sense.
<!-- more -->

## Column-by-Column Testing

I also tested generating **one column at a time** to check if a columnar approach would be more efficient. Still evaluating the results, but it's looking promising.

## Prompt Quality: All-at-once vs Focused Text Generation

When asking the model to generate all columns at once — including text — the results weren’t great. The **text column was often too short** (asked for 130 words, got ~90 on average), and the **quality** wasn’t great either.

However, when using a **dedicated prompt just for the text column**, the output improved a lot. Even if the model delivered 106–108 words when 120 were requested, the quality and coherence were way better. This small deviation is totally fine — if the text makes sense and it's close to the target, that’s good enough.

## Speed: One-by-One vs Batch Generation

Generating individual rows takes around **2.5 to 3.5 seconds each**, while doing it in **batches of 30** brings it down to **0.8 to 1.2 seconds per row**. Big difference.

I think this is related to how some models "think" during generation — when done in batch, the token prediction seems more optimized.

## Agents and Tools: When More Is Less

When using agents with tools, setting temperature to **0** made things worse — the agent got stuck trying to figure out which tool to use. With **0.7**, things moved faster, but the agent still wouldn't call any tool directly. Instead, it just outputs something like:

```python
call_tool({"param": "value"})
```

…without actually calling the tool.

Turns out, having **too many tools** available causes confusion. I’m now testing agents with **fewer tools** each to see if it improves performance and reliability.

## Prompt Fix: Mention the Metadata Clearly

One big issue was that the prompt only referenced the metadata in the **user input**, not in the **system instructions**. Once I explicitly told the agent to use the metadata in the system prompt, the success rate jumped: in **30 consecutive runs**, it correctly used the tool every single time.

## Problem: Generic + Specific Tool in the Same Node

When you put a **generic tool and a specific one** in the same node, even if you tell the model to prioritize the specific one, it sometimes uses both — which defeats the purpose. This seems to be a limitation and needs to be handled better at the agent level.

## Switching to Qwen3-14B Made a Big Difference

After switching from a smaller model to **Qwen3-14B**, the tool usage accuracy improved a lot. Much more reliable.

## What’s Next

Still need to dig deeper into how to **improve text generation** — might be a model limitation, especially when trying to control output length and coherence at the same time.

Also looking into fine-tuning **chat parameters** like `keep_alive` and setting limits for predictions. Sometimes the model starts rambling or looping, so having tighter constraints might help stabilize output.


