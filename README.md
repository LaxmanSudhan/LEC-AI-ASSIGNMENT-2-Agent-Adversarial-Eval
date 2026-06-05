# LEC-AI-ASSIGNMENT-2-Agent-Adversarial-Eval

# Assignment 2 - Agent, Adversarial Eval

## Why I Picked This Assignment

At Mastercard, I learned that the hardest part of shipping LLM systems isn't building them — it's **trusting them not to break in production**.

We had a compliance pipeline that ran every quarter. Before LLMs, analysts manually reviewed 2,800+ scenarios. After we added a RAG layer, we cut manual work by 93%. But then a new problem appeared: the model would sometimes "helpfully" answer questions it shouldn't — like hallucinating a compliance rule that didn't exist.

My manager didn't care about accuracy on happy paths. He cared about **what happens when the model doesn't know**. Because in compliance, a confident wrong answer is worse than an "I don't know."

This assignment is exactly that problem: build an agent that chooses tools, survives failures, and gets punished for hallucinating. That's real. That's what I debugged at Mastercard.

So I picked this assignment not because it's easy — but because I've lived the failure mode it's testing.

---

## How I Approached Each Requirement

| Requirement | My Decision | Why |
|-------------|-------------|-----|
| **3 meaningfully different tools** | Calculator (compute), NoteStore (memory), FactLookup (knowledge) | At Mastercard, our tools were distinct: compliance DB (facts), user profile store (memory), and a calculator for fee math. If tools overlap, the LLM never has to reason. |
| **At least one stateful** | SQLite-backed NoteStore | Real persistence = real failures. I've seen production agents crash because a DB connection dropped mid-conversation. I wanted to test that. |
| **LLM-driven tool selection** | Grok API, `tool_choice="auto"`, zero if-statements | No keyword matching. The LLM reads tool descriptions and decides. At Mastercard, we banned hardcoded routing because compliance rules change too fast. |
| **Graceful degradation** | `execute_tool()` catches every exception type | Never raises. Returns `(string, bool)`. The agent continues after division by zero, DB lock, timeout. This is non-negotiable in production. |
| **Deliberate failures in eval** | 16 raw failures + 5 agent failures + chained recovery | If you don't test the crash, it will crash. I learned this when our RAG pipeline silently failed for six hours. |
| **20 prompts (happy/ambiguous/OOS)** | Designed overlaps that are plausible, not obvious | At Mastercard, the ambiguous cases were the dangerous ones: "is this a compliance lookup or a user memory question?" I wanted the LLM to struggle with that boundary. |
| **Two system prompts** | A = conservative (exact refusal), B = exploratory (prefer tools) | I wanted to see if strictness helps or hurts. In production, we defaulted to conservative because compliance. But I suspected it slowed things down. |
| **Report accuracy, abstention, latency** | Weighted scoring (40% acc, 35% abstention, 25% speed) | Accuracy is table stakes. Abstention prevents hallucination. Speed matters at scale (140 analyst hours saved). |
| **Pick one to ship** | Prompt B (faster, equally accurate) | Tied on correctness. Speed broke the tie. That's a real operational decision. |
| **Name a failure mode** | Agent calls `fact_lookup` on "tell me something interesting" | Honest documentation. At Mastercard, we logged every failure. "Worked perfectly" would have gotten my PR rejected. |

---

## The Work LLMs Cannot Do (And Why I Had To)

| Task | Why LLM Fails | What I Did |
|------|----------------|-------------|
| Choose three tools | LLM picks three search variants to avoid hard decisions | I picked compute, memory, knowledge — real overlaps that force reasoning |
| Design ambiguous prompts | LLM generates obvious ambiguity ("math or memory?") | I wrote prompts where the correct *first tool* depends on understanding the user's intent |
| Build deliberate failures | LLM avoids breaking things because it wants to look good | I added division by zero, timeout simulation, permission errors — things that actually happen in production |
| Justify shipping decision | LLM says "both are good, choose based on needs" | I picked B based on weighted score: 0.017s faster at the same accuracy. That's a real ops call. |
| Name a failure mode | LLM says "system performed excellently" (disqualifying answer) | I documented fact_lookup overgeneralization with a concrete example: "Tell me something interesting" |

---

## What I Learned From Mastercard That Applied Here

**1. Abstention is a feature, not a bug.**  
Our compliance agent refused 15% of queries. That was fine. Hallucinating would have been a regulatory incident.

**2. Latency compounds.**  
Saving 0.02 seconds per call across 2,800 scenarios? That's a minute. Not huge. But across a quarter? The analysts noticed.

**3. Failure modes are inevitable. Document them.**  
We kept a live log of every hallucination. This assignment demands the same honesty.

---
---

## One Line Summary

**I built an agent that chooses tools via LLM reasoning (no if-statements), survives 22 deliberate failure scenarios, scored 100% on 20 eval prompts, and shipped Prompt B (faster); known failure: fact_lookup fires on vague factual-sounding prompts instead of abstaining.**
