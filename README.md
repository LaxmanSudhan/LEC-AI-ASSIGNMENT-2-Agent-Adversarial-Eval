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
