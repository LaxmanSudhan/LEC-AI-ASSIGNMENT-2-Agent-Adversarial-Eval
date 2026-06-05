# LEC-AI-ASSIGNMENT-2-Agent-Adversarial-Eval
# Assignment 2 - Agent, Adversarial Eval

## Why I Picked This Assignment

I worked at Mastercard on an LLM-driven evaluation system. That experience taught me that most AI evaluations are not effecient — because they only test what the model is good at.

At Mastercard, I built pipelines that processed compliance data. We thought the system was solid. Then we actually stressed it. It hallucinated deadlines. It picked the wrong retrieval tool. It crashed silently.

From my experience, I learned that An LLM can generate code. An LLM can write documentation. But an LLM cannot:

- **Design meaningful ambiguity** — It picks obvious overlaps, not subtle ones that actually trick the agent.
- **Build deliberate failures** — It avoids breaking things because that makes the system look bad.
- **Make trade-off decisions** — It gives you both options. I have to pick one and justify it.


This assignment asks exactly that to build an agent, then deliberately try to fail it.

### Why Not the Other Two Assignments

There were three assignments to choose from. I looked at all of them.

| Assignment | What It Asked | Why I Said No |
|------------|---------------|----------------|
| **Assignment 1** | Build a chatbot with memory and personality | A bit vague. "Personality" is subjective. Hard to measure success or failure. Felt like product design, not engineering evaluation. |
| **Assignment 2 (my choice)** | Build an agent with tools, adversarial eval, mandatory failure mode | Clear requirements. Explicitly punishes hallucination. Forces honesty about where it breaks. This is real evaluation work. (More interesting to me) |
| **Assignment 3** | Fine-tune a small model on a custom dataset | Fine-tuning is valuable, but it's a one-time task. This assignment is about building a *system*: tools, agent, harness, comparison. |

I picked this assignment because it rewards honesty about failure — something I learned to value at Mastercard, and something most AI evaluations actively avoid.


## The Decisions I Made and the Alternatives I Ruled Out

Throughout this assignment, I made several deliberate choices. For each, I considered alternatives and rejected them for specific reasons.

---

### Decision 1: Tool Selection

**What I chose:** Calculator (stateless math), NoteStore (stateful SQLite memory), FactLookup (stateless knowledge base)

**Alternatives I ruled out:**

| Alternative | Why I rejected it |
|-------------|-------------------|
| Three search variants (web, internal, vector) | Violates "meaningfully different" requirement. The LLM wouldn't need to discriminate — any tool would work. |
| Weather API + Stock API + News API | Too similar (all external APIs). Also requires live internet, making deterministic testing impossible. |
| File reader + Email sender + Calendar | Over-engineered. SQLite gives me stateful behavior without external dependencies. |

**My reasoning:** I wanted tools that force the LLM to actually *think* about which one fits. Compute vs memory vs knowledge is a clean separation. The ambiguity comes naturally (e.g., "20% of my budget" needs memory THEN compute).

---

### Decision 2: LLM Model

**What I chose:** Grok-3-mini (xAI API)

**Alternatives I ruled out:**

| Alternative | Why I rejected it |
|-------------|-------------------|
| GPT-4 | Expensive for 40 eval calls. Also too capable — would hide failure modes that cheaper models expose. |
| GPT-3.5 | Cheaper but function calling is less reliable. I tested and got inconsistent tool selections. |
| Local LLM (Llama 3) | Too slow. Latency would make the eval impractical (10+ seconds per turn). |
| Claude | No Grok-like rate limits. xAI gave me 1000 free credits. |

**My reasoning:** Grok-3-mini is cheap, fast enough (~2s per turn), and its function calling works. The assignment doesn't specify a model, so choosing a cost-effective one is a valid engineering trade-off.

---

### Decision 3: System Prompt Design

**What I chose:** Two distinct prompts — A (conservative, strict refusal) and B (exploratory, prefers tools)

**Alternatives I ruled out:**

| Alternative | Why I rejected it |
|-------------|-------------------|
| One prompt, run once | Assignment explicitly requires comparing two prompts. |
| A vs B with tiny differences (e.g., wording only) | Wouldn't produce meaningful comparison. My prompts have real philosophical differences. |
| Three prompts | Two is sufficient. More would just add noise. |

**My reasoning:** Prompt A forces exact refusal phrasing. Prompt B allows conversational responses. I wanted to see if strictness improves accuracy or just slows things down. Turns out — same accuracy, Prompt B is faster.

---

### Decision 4: Graceful Degradation Test Structure

**What I chose:** Three-layer test suite (raw tool failures → agent loop failures → chained recovery)

**Alternatives I ruled out:**

| Alternative | Why I rejected it |
|-------------|-------------------|
| Only test raw tool failures | Doesn't prove the agent loop survives. The wrapper could catch exceptions but the agent could still crash. |
| Only test happy paths | Missing the entire point of graceful degradation. |
| Mock failures instead of real ones | Mocking doesn't prove real exceptions are caught. I used actual division by zero, DB permission errors, and timeout simulations. |

**My reasoning:** The assignment says "build deliberate failures into your eval." I built 22 of them across three layers. The chain test (fail mid-conversation, then recover) is the most realistic — and it passed.

---

### Decision 5: Evaluation Harness Prompt Design

**What I chose:** 20 prompts split into happy path (10), ambiguous (5), out-of-scope (5)

**Alternatives I ruled out:**

| Alternative | Why I rejected it |
|-------------|-------------------|
| More happy path, fewer ambiguous | Wouldn't test the hard cases. The assignment cares about ambiguous routing and abstention. |
| All prompts independent (no shared history) | Would miss testing stateful memory across turns (HP-05 → HP-08). |
| Ambiguous with only one correct answer | That's just happy path. Ambiguous needs *two plausible tools*. |

**My reasoning:** The ambiguous prompts are the most interesting. AM-01 ("20% of my budget") could use note_store (read budget) or calculator (compute percentage). I preferred note_store first because the computation has no input without memory. The LLM agreed every time.

---

### Decision 6: Scoring Weights for Shipping Decision

**What I chose:** 40% accuracy, 35% abstention, 25% speed

**Alternatives I ruled out:**

| Alternative | Why I rejected it |
|-------------|-------------------|
| Equal weights (33/33/33) | Accuracy matters more than speed for a production agent. |
| Speed as tie-breaker only | Didn't work here — accuracy was tied, so speed became the decider naturally. |
| No weights, just pick | Arbitrary. The assignment asks for justification. Weights make the decision transparent. |

**My reasoning:** A hallucinated answer is worse than a slow correct answer. So accuracy > abstention > speed. But speed still matters. My weights reflect that.

---

### Decision 7: FactLookup Not-Found Behavior

**What I chose:** Return `{"found": False, "result": None, "error": "..."}` instead of hallucinating

**Alternatives I ruled out:**

| Alternative | Why I rejected it |
|-------------|-------------------|
| Raise an exception for not-found | Would treat missing facts as errors, not abstention. The agent might crash. |
| Guess an answer (hallucinate) | Assignment explicitly punishes this. "Worked perfectly" is not a valid finding. |
| Return empty string | Same as None but less explicit. The LLM might ignore it and guess. |

**My reasoning:** Returning None forces the agent to handle "no fact found" explicitly. The LLM sometimes guesses anyway (my documented failure mode), but the tool itself doesn't hallucinate.

---

### Decision 8: SQLite for Stateful Storage (Not JSON)

**What I chose:** SQLite database with persistent disk storage

**Alternatives I ruled out:**

| Alternative | Why I rejected it |
|-------------|-------------------|
| In-memory Python dict | Wouldn't survive across process restarts. Also no real failure modes (permissions, corruption). |
| JSON file read/write | Works, but less robust for concurrent access. SQLite gives me real OperationalError to test graceful degradation. |
| No stateful tool | Assignment requires at least one. NoteStore is my only stateful tool, so it had to work. |

**My reasoning:** SQLite gives me real failure modes (NOTE-F4: `/root/no_permission.db` raises RuntimeError). The agent must catch this and continue. A JSON file would just fail differently.

---

## One Table Summary

| Decision | What I Chose | What I Ruled Out | Why |
|----------|--------------|------------------|-----|
| Tools | Calculator, NoteStore, FactLookup | Three search variants | Forces real discrimination |
| Model | Grok-3-mini | GPT-4, GPT-3.5, Claude | Cheap, fast, good function calling |
| System prompts | A (strict) vs B (exploratory) | Tiny wording changes | Real philosophical difference |
| Degradation test | 3 layers (22 scenarios) | Happy path only | Tests actual crashes |
| Eval prompts | 20 (10/5/5 split) | All independent | Tests stateful memory across turns |
| Shipping weights | 40% acc / 35% abstain / 25% speed | Equal weights | Accuracy matters most |
| Not-found behavior | Return None, no hallucination | Guess or raise exception | Assignment punishes hallucination |
| Stateful storage | SQLite | JSON file, dict | Real failure modes (permissions) |

---

## One Line Takeaway

I chose tools, prompts, and tests that create real ambiguity and failure modes — because the assignment rewards honesty about breaking, not pretending everything worked perfectly.

