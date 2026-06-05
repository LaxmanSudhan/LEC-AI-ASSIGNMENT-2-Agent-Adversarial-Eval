# LEC-AI-ASSIGNMENT-2-Agent-Adversarial-Eval

## Part 1: Tool Design & Justification
## The Three Tools I Built

I didn't overthink this. I picked tools that are obviously different so the LLM has to actually think about which one to use.

| Tool | What It Does | Why I Chose It |
|------|--------------|----------------|
| **Calculator** | Basic math (add, multiply, sqrt, etc.) | Pure numbers. No memory. No facts. Just math. |
| **NoteStore** | Remembers things across conversations (SQLite) | This is my stateful one. Saves whatever you tell it. |
| **FactLookup** | Answers factual questions from a hardcoded list | Static knowledge. Capitals, science stuff, history. |

## Why They're Not "Three Search Tools"

If every tool just searches for stuff, the LLM doesn't have to make real decisions. Here's the real difference:

- **Calculator** → "What's 2+2?" (don't search for this, just compute it)
- **NoteStore** → "Remember my name is John" (this isn't a fact, it's user-provided)
- **FactLookup** → "What's the capital of France?" (this IS a fact, look it up)

## Where I Deliberately Break Things

I added specific failure triggers to test graceful degradation. The agent has to keep running, not crash.

| Tool | When It Breaks | What Happens |
|------|----------------|---------------|
| Calculator | You try to divide by zero | Raises `ZeroDivisionError` |
| NoteStore | You give it a bad database path (like `/root/no_permission.db`) | Raises `RuntimeError` |
| FactLookup | You ask for "simulate_timeout" | Raises `ConnectionError` |

Why exceptions instead of error messages? Because if I just return `{"error": "..."}`, the agent could ignore it. Exceptions force the agent to actually handle failures.

## What The LLM Couldn't Do For Me

If I asked ChatGPT to design these tools, it would give me three search tools or three APIs that all work nicely. It wouldn't create real ambiguity or genuine failure modes because that makes the LLM look bad.

I had to design the hard cases myself. That's the actual work.

## Quick Test Results

I ran the smoke test. Everything works as expected:

- Calculator: `2+2=4`, `10/0` → error (good)
- NoteStore: Saved "budget = £500", read it back, deleted it (all good)
- FactLookup: Found Tokyo, Paris, Shakespeare. "best pizza topping" → None (good, no hallucination). "simulate_timeout" → ConnectionError (good, tests graceful failure)

# Part 2: LLM-Driven Tool Selection

## What I Built

An agent that uses **Grok (xAI)** to decide which tool to call. No if-statements. No keyword routing. The LLM reads tool descriptions and figures it out.

## How I Know if its FAIR:

My code has exactly **zero** of these patterns:
- `if "math" in prompt`
- `if "remember" in prompt`  
- `if "capital" in prompt`

The only decision logic is `tool_choice="auto"`. Grok decides. I just execute what it picks.

## The Agent Loop

User prompt → Grok decides (tool or abstain) → Execute tool → Return result

## Experimental Decisions I Made

### Decision 1: Two System Prompts

| Prompt | Philosophy | Why |
|--------|------------|-----|
| **A (Conservative)** | "Always use a tool if it fits. If not, exact refusal phrase." | Tests whether strict rules improve accuracy |
| **B (Exploratory)** | "Prefer tools. When uncertain, try anyway." | Tests whether flexibility helps ambiguous cases |

I'll compare these in the evaluation harness.

### Decision 2: Grok-3-mini over GPT-4

Why? Because the assignment doesn't specify a model, and Grok's function calling is solid. Also, xAI's API rate limits are generous for testing.

### Decision 3: Explicit Tool Error Messages

When a tool fails (division by zero, DB error, timeout), the agent returns a structured error:

`TOOL ERROR (ZeroDivisionError): Division by zero is undefined.`

The LLM sees this and responds naturally. The agent never crashes.

