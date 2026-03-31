# Attractor as a Design Iteration Tool

**Date:** 2026-03-31
**Status:** Exploration
**Spec:** https://github.com/strongdm/attractor

## Idea

Use [attractor](https://github.com/strongdm/attractor) — a DOT-based pipeline runner for multi-stage AI workflows — to prototype and iterate on chuck's agentic loop before implementing it in Go/Temporal.

The agent loop's architecture (workflow shape, activity contracts, state model) is well-defined. What's *not* defined are the pieces that require empirical iteration with real LLM calls:

1. **System prompt and context assembly** — what goes in, in what order, with what framing
2. **Tool result formatting** — JSON? Markdown? Truncated how?
3. **Compaction summarization prompt** — how to compress middle turns without losing critical detail
4. **Budget exhaustion behavior** — does "strip tools and force final" actually produce useful responses?
5. **Fidelity/context strategy** — full history vs. summarized vs. truncated, and when to switch
6. **Iteration limits** — are 10 tool calls and 5 LLM iterations the right defaults?

## Why Attractor Fits

Attractor's DOT graphs map naturally to chuck's loop:

| Chuck concept | Attractor concept |
|---------------|-------------------|
| AgentIteration activity | `box` node with prompt |
| ExecuteTool activity | `parallelogram` (tool) node |
| "done" vs "needs_tools" routing | `diamond` conditional with edge conditions |
| Budget check / force final | Conditional edge + separate node with stripped tools |
| Compaction | `box` node with `class="cheap"` (haiku) |
| Context fidelity | `fidelity` attribute (full/truncate/compact/summary) |
| Retry policies | `max_retries` + backoff presets |

Changing a prompt, routing condition, or fidelity mode is a one-line DOT edit followed by a re-run — no Go compilation, no Temporal worker restart.

## What the POC Graph Covers

See `2026-03-31-attractor-agent-loop.dot` — a single-turn agent loop with:

- Context assembly (system prompt + skill directory + history + tool results)
- Agent iteration with budget-aware prompting
- Conditional routing: done → publish, needs_tools → budget check → execute or force final
- Tool execution with result formatting
- Post-turn compaction check and summarization
- Failure routing for LLM errors and tool errors
- Model stylesheet: sonnet for reasoning, haiku for cheap ops (progress, compaction)

## What to Iterate On

Run the graph against real queries and document corpora. Focus on:

### Round 1: System Prompt
- Does the LLM understand its role and available tools?
- Does it search before answering or guess?
- Does it use multiple tools for complex queries?
- Does the skill directory format work?

### Round 2: Tool Results
- Is the result formatting clear enough for the LLM to synthesize?
- Does truncation at 4000 tokens lose critical information?
- Does the LLM request follow-up searches when results are truncated?

### Round 3: Budget Behavior
- What happens at budget exhaustion? Useful partial answers or confused output?
- Is the force-final prompt effective?
- Are 10 tool calls / 5 iterations the right defaults, or too conservative?

### Round 4: Compaction
- Does the summarization prompt preserve source citations?
- Can the LLM reason effectively over compacted history in the next turn?
- What's the right trigger threshold (size? turn count? both)?

### Round 5: Fidelity Modes
- When should we switch from `full` to `compact`?
- Does `summary:medium` lose too much detail for follow-up questions?
- Is there a sweet spot between context size and answer quality?

## Path to Temporal

Once the graph stabilizes — prompts are proven, routing is validated, thresholds are tuned — translate to Go/Temporal. The translation is mechanical:

- Each `box` node's prompt becomes a constant in the activity package
- Routing conditions become `if` statements in `processTurn`
- Fidelity/compaction thresholds become config values with known-good defaults
- The model stylesheet becomes `AgentConfig` fields

An LLM can assist with this translation given a few hand-written examples as reference, since the mapping between attractor concepts and Temporal patterns is regular.

## Not Covered

This graph models a **single turn**. Not modeled:

- Multi-turn conversation flow (signal handling, continue-as-new)
- Subagent spawning and result integration
- Skill loading mid-conversation
- Auth and API layer
- DynamoDB/S3 persistence (attractor uses its own artifact store)

These are architectural concerns already well-specified in the existing design docs. The attractor graph focuses on the empirical, prompt-engineering-heavy inner loop.
