---
draft: false
date: 2025-06-27
categories:
  - Datastack
  - LLM
tags:
  - LLM
  - ollama
  - langgraph
  - homelab
  - datastack
  - pandas
  - clickhouse
  - hyperdx
  - clickstack
---
# Exploring Structured Data Generation with LLMs - Part 2

## Identified Issues

While working on generating structured data with LLMs, I encountered a few recurring challenges:

- JSON is not always returned in a valid format — in some cases, the model fails to structure it correctly.
- CSV output often includes additional messages, breaking automated parsing.
- The quality of generated CSVs is still not satisfactory.
- Assessing "quality" is subjective. My criteria include: whether randomness makes sense in certain columns (e.g., names), if patterns like IDs are followed, and if textual fields are coherent.

## Format Adjustments

I'm trying to standardize the output format for both JSON and CSV. To achieve this, I started implementing **guardrails** — mechanisms that enforce the structure expected from the model.

### JSON

The **Qwen3** model often wraps responses in a `"the thing"` tag, which requires post-processing. Now, using *parsers*, this is handled automatically.

An example of an improperly formatted output:

```shell
Okay, let's tackle this again...
</think>

{
  "theme": "user register",
  "columns": 15,
  "rows": null,
  "specification": ""
}
```

To enforce correct formatting, I added specific instructions to the prompt such as:

```yaml
- Wrap the JSON output inside a code block formatted as ```json (three backticks followed by "json").
```

Actual data generation (simulated data) will be covered in a future iteration.

## Testability with LLMs

As someone who appreciates unit testing, I realized it's feasible in this context. Each **function** (or **graph node**) can be tested individually since I know what to expect from each step.

One concern is the risk of **prompt injection** or other unexpected behaviors. I'm considering using a second LLM to try to "break" the system and identify vulnerabilities.

## Code Organization

To support better testing and modularity, I split the code into smaller files. For example, the `agent` file is now only responsible for creating the LangGraph graph.

### CSV Parser

I couldn't find a CSV parser that suited my needs (parsing markdown blocks), so I built a custom one. It still has room for improvement but can already extract content properly from markdown code blocks like:

````markdown
```csv
name,age
alice,30
```
````

## Observability with OpenLIT and the ClickHouse + HyperDX Stack

To better understand the LLM behavior, especially for more complex runs, I needed tracing beyond simple `print()` statements. That's when I discovered [**OpenLIT**](https://github.com/openlit/openlit), which turned out to be a pleasant surprise.

With just:

```python
import openlit
openlit.init()
```

it starts capturing model interactions automatically — including Ollama, Qwen, LangChain, LangGraph.It logs:

- LLM inputs and outputs,
- token usage,
- estimated cost,
- latency,
- and GPU usage, if available.

All using **OpenTelemetry**, which means I can export data to any OTLP-compatible backend. This fits perfectly with my stack using **ClickHouse** and **HyperDX**, also known as **ClickStack**.
<!-- more -->

### ClickStack

**ClickStack** is an observability stack built on:

- **ClickHouse** as the analytical engine for logs, metrics, and events;
- **HyperDX** as the interface for logs, traces, metrics, alerts, session replay, etc;
- and an **OpenTelemetry Collector** with a pre-configured, opinionated setup for telemetry ingestion.

This combo allows:

- fast local deployment (Docker Compose) or Helm for Kubernetes;
- querying through the UI or SQL (ClickHouse) or Lucene (HyperDX);
- detailed traces for each function in the pipeline;
- and native integration with tools like OpenLIT.

### Why It Matters

With each graph node isolated, I can **test individual steps**, trace the full lifecycle of messages, measure response time, and even check token usage. This is crucial when working with models like Qwen3, which can produce inconsistent or unpredictable responses.

OpenLIT integrates smoothly with ClickStack. It even supports direct OTLP export to ClickHouse, making it easy to visualize function traces.

## A Specific Case: Entity Extraction

One of the most complex nodes I've tested is the entity extractor. It needs to identify:

- Dataset theme,
- Number of columns,
- Number of rows,
- Additional specifications if present.

To validate this, I crafted prompts with various intentions. In most cases, the node performed well, especially for extracting specific fields. However, in more open-ended prompts, it occasionally captured parts of the original prompt — a typical symptom of *prompt injection*.

## Input Sanitization

To handle this, I'm exploring ways to **sanitize** content before passing it forward. Options include:

- Using another LLM to validate output (more robust, but costly),
- Implementing simple manual checks (e.g., looking for suspicious keywords) to avoid extra inference calls.

[github](https://github.com/kleber-yokota/agent_data_generator)