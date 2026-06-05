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

# Part 3: Graceful Degradation

## What I Tested

I built a test suite that deliberately breaks every tool in every possible way to prove the agent never crashes.

**Three layers of testing:**

| Part | What It Tests | Number of Scenarios |
|------|---------------|---------------------|
| **A** | Raw tool failures via `execute_tool()` | 16 |
| **B** | Full agent loop with bad prompts | 5 |
| **C** | Chained failure + recovery mid-conversation | 5 turns |

## Part A — Raw Tool Failures (16 tests)

I fed garbage inputs directly to `execute_tool()` and verified it returns `(string, False)` instead of raising exceptions.


**Result:** 16/16 passed. `execute_tool()` never raised. Agent would not crash on any of these.

## Part B — Agent Loop Failures (5 tests)

I sent bad prompts through the full `run_agent_turn()` loop and checked:
- No traceback or crash words in response
- Agent error is `None` (loop didn't blow up)


**Result:** 5/5 passed. The agent responded gracefully every time. No tracebacks. No crashes.

**What I noticed:** The agent's wording varies. For division by zero it said "Division by zero is undefined." For malformed expression it abstained entirely. Still graceful, just different styles.

## Part C — Chained Failure + Recovery

The most important test. A realistic conversation where a tool fails mid-session, then the agent must keep working.

**Conversation flow:**

| Turn | Prompt | Expected | Tool Success | Result |
|------|--------|----------|--------------|--------|
| 1 | "Remember my name is Jordan" | note_store | ✓ | Name saved |
| 2 | "What is 144 divided by 12?" | calculator | ✓ | 12 |
| 3 | "What is 7 divided by 0?" | calculator | ✗ FAILS | Graceful error |
| 4 | "What is my name?" | note_store | ✓ | "Jordan" (state intact) |
| 5 | "What is the capital of Germany?" | fact_lookup | ✓ | "Berlin" |

**Result:** All 5 turns passed. The agent survived a tool failure and continued normally. SQLite state persisted across the failure.

# Part 4: Evaluation Harness

## What I Built

A 20-prompt evaluation harness that tests the agent on three categories:

| Category | Count | Expected |
|----------|-------|----------|
| Happy Path | 10 | Correct tool selection |
| Ambiguous | 5 | Acceptable tool from allowed set |
| Out of Scope | 5 | Abstain (no tool called) |

---

## Experimental Decisions

| Decision | Why |
|----------|-----|
| **Fresh history per prompt** (except HP-05/HP-08) | Isolates tests so one failure doesn't cascade |
| **Shared history for HP-05 → HP-08** | Tests real stateful memory (save then recall deadline) |
| **Ambiguous accepts multiple tools** | LLM can choose either; both pass if justified |
| **Out of scope = zero tool calls** | Any tool call = hallucination attempt = FAIL |
| **Latency tracked per prompt** | Measures practical performance |

---

## Evaluation Results (System Prompt A)

| Metric | Result |
|--------|--------|
| Tool selection accuracy | 10/10 (100%) |
| Ambiguous accuracy | 5/5 (100%) |
| Abstention rate | 5/5 (100%) |
| Overall accuracy | 20/20 (100%) |
| Mean latency | 2.258 seconds |

---

### Key Finding

The agent scored **perfectly** on all 20 prompts with System Prompt A:

- No wrong tools on happy path
- No hallucination on out-of-scope prompts
- Stateful memory (HP-05 → HP-08) worked correctly

---

### One Nuance

The ambiguous cases all matched the preferred tool. The LLM never needed to use the fallback acceptable tool. 

This is good but means I didn't test the edge where both tools would be acceptable — the LLM always picked my preferred choice, so I never saw how it would behave when truly torn between two valid options.

# Part 5: System Prompt Comparison

## What I Did

Ran the same 20 evaluation prompts twice — once with **System Prompt A (Conservative)** and once with **System Prompt B (Exploratory)** — then compared results to decide which to ship.

---

## Experimental Decision

| Decision | Why |
|----------|-----|
| Weighted scoring (accuracy 40%, abstention 35%, speed 25%) | Values correctness over speed, but speed still matters |
| Normalised latency for comparison | Lower latency = better, converted to score |
| Ties allowed in individual metrics | Overall weighted score determines winner |

---

## Comparison Results

| Metric | Prompt A | Prompt B | Winner |
|--------|----------|----------|--------|
| Tool-selection accuracy | 100% | 100% | TIE |
| Ambiguous routing accuracy | 100% | 100% | TIE |
| Out-of-scope abstention | 100% | 100% | TIE |
| Overall accuracy | 100% | 100% | TIE |
| Mean latency | 2.166s | 2.149s | **B WINS** |

---

## Weighted Scores

| Prompt | Weighted Score |
|--------|----------------|
| A (Conservative) | 0.7501 |
| B (Exploratory) | 0.7521 |

---

## Shipping Decision

**Ship Prompt B (Exploratory)**

**Why:** Both prompts scored perfectly on accuracy and abstention. The tie-breaker was latency — Prompt B was slightly faster (2.149s vs 2.166s). The exploratory prompt's "prefer tools but be conversational" phrasing didn't hurt correctness but produced marginally faster responses.

---

## What I Learned

Both prompts performed identically on correctness — 20/20 correct across all categories. The conservative vs exploratory distinction didn't matter for these specific prompts. A harder ambiguous set might have shown a difference.

---

## Raw Run Data

| Category | Prompt A | Prompt B |
|----------|----------|----------|
| Happy path correct | 10/10 | 10/10 |
| Ambiguous correct | 5/5 | 5/5 |
| Out-of-scope correct | 5/5 | 5/5 |
| Total correct | 20/20 | 20/20 |
| Mean latency | 2.166s | 2.149s |

# Part 6: Shipping Decision & Failure Mode

## Shipping Decision

**Ship Prompt B (Exploratory)**

### Justification

Both prompts scored perfectly on all 20 prompts. The decision comes down to **latency** — Prompt B is 0.017 seconds faster on average. In production, marginal speed differences matter at scale.

**Why not Prompt A?** The conservative prompt's stricter phrasing ("respond ONLY with exact phrase") adds no accuracy benefit but forces slightly longer processing. Prompt B's conversational style is equally correct and faster.

---

## Observed Failure Mode

**Failure:** Agent calls `fact_lookup` for queries that are not encyclopaedic facts.

### Real Example from Testing

| User Prompt | What Agent Did | What It Should Have Done |
|-------------|----------------|--------------------------|
| "Tell me something interesting" | Called `fact_lookup(query="interesting fact")` → returned nothing | Abstain or respond directly |

### Why This Happens

The `fact_lookup` description says "capitals, science, history, definitions". The LLM overgeneralizes "interesting fact" as a factual request when it's actually a vague open-ended prompt. No tool fits — the agent should abstain but tries anyway.

### Impact

- Lowers abstention rate on vague factual-sounding requests
- Wastes API call (fact_lookup returns nothing)
- Creates awkward user experience ("I found nothing")

### Attempted Mitigation

Added "encyclopaedic facts only" to the tool description in Prompt B. Reduced failure rate from ~30% to ~15% — still not eliminated.


---

## One Line Summary

**Ship Prompt B (faster, equally accurate); known failure: agent calls fact_lookup on vague factual-sounding prompts instead of abstaining.**

