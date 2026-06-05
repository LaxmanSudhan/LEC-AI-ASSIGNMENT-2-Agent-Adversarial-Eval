# LEC-AI-ASSIGNMENT-2-Agent-Adversarial-Eval

### NOTE: 
1. Please view my **detailed explantion of my approach** in the **README file** uploaded inside the soultion folder named "LEC-AI-ASSIGNMENT-2-Agent-Adversarial-Eval".
2. **my google colab link : https://colab.research.google.com/drive/1ZW7cOiURZ3m7LLyA4eAYBf0wpckYdf4b?usp=sharing**
3. Please note that I have not included the xAI API key in the repository, as API credentials are confidential and should not be shared according to terms of conditions. The key was stored locally via environment variables during development. Please see below on how to insert key in colab notebook.

**FOR API KEY :**

To run the project:

-> Obtain an API key from xAI.

-> In Google Colab, open Secrets (key icon in the left sidebar).

-> Create a new secret named GROK_API_KEY and paste your API key as the value.

-> Save the secret and ensure notebook access is enabled.

-> Run the notebook/project normally. The code will automatically read the key from the GROK_API_KEY secret.
# -> Why I Picked This Assignment

I worked at Mastercard on an LLM-driven evaluation system. That experience taught me that most AI evaluations are not effecient because they only test what the model is good at.

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


# -> The Decisions I Made and the Alternatives I Ruled Out

Throughout this assignment, I made several deliberate choices. For each, I considered alternatives and rejected them for specific reasons.

---

### Decision 1: Tool Selection

**What I chose:** Calculator (stateless math), NoteStore (stateful SQLite memory), FactLookup (stateless knowledge base)

**Alternatives I ruled out:**

| Alternative | Why I rejected it |
|-------------|-------------------|
| Three search variants (web, internal, vector) | Violates "meaningfully different" requirement. The LLM wouldn't need to discriminate  any tool would work. |
| Weather API + Stock API + News API | Too similar (all external APIs). Also requires live internet, making deterministic testing impossible. |
| File reader + Email sender + Calendar | Over-engineered. SQLite gives me stateful behavior without external dependencies. |

**My reasoning:** I wanted tools that force the LLM to actually *think* about which one fits. Compute vs memory vs knowledge is a clean separation. The ambiguity comes naturally (e.g., "20% of my budget" needs memory THEN compute).

---

### Decision 2: LLM Model

**What I chose:** Grok-3-mini (xAI API)

**Alternatives I ruled out:**

| Alternative | Why I rejected it |
|-------------|-------------------|
| GPT-4 | Expensive for 40 eval calls. Also too capable would hide failure modes that cheaper models expose. |
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

**My reasoning:** Prompt A forces exact refusal phrasing. Prompt B allows conversational responses. I wanted to see if strictness improves accuracy or just slows things down. Turns out same accuracy, Prompt B is faster.

---

### Decision 4: Graceful Degradation Test Structure

**What I chose:** Three-layer test suite (raw tool failures → agent loop failures → chained recovery)

**Alternatives I ruled out:**

| Alternative | Why I rejected it |
|-------------|-------------------|
| Only test raw tool failures | Doesn't prove the agent loop survives. The wrapper could catch exceptions but the agent could still crash. |
| Only test happy paths | Missing the entire point of graceful degradation. |
| Mock failures instead of real ones | Mocking doesn't prove real exceptions are caught. I used actual division by zero, DB permission errors, and timeout simulations. |

**My reasoning:** The assignment says "build deliberate failures into your eval." I built 22 of them across three layers. The chain test (fail mid-conversation, then recover) is the most realistic and it passed.

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

I chose tools, prompts, and tests that create real ambiguity and failure modes because the assignment rewards honesty about breaking, not pretending everything worked perfectly.


# -> Headline Numbers

Here are the results from running the full evaluation with both system prompts.

---

### Summary Table

| Metric | Prompt A (Conservative) | Prompt B (Exploratory) |
|--------|------------------------|------------------------|
| Happy path accuracy | 10/10 (100%) | 10/10 (100%) |
| Ambiguous routing accuracy | 5/5 (100%) | 5/5 (100%) |
| Out-of-scope abstention | 5/5 (100%) | 5/5 (100%) |
| Overall accuracy (20 prompts) | 20/20 (100%) | 20/20 (100%) |
| Mean latency (seconds) | 2.166s | 2.149s |
| Weighted score | 0.7501 | 0.7521 |

Both prompts achieved perfect accuracy across all 20 evaluation prompts; Prompt B was marginally faster.

---

### Graceful Degradation (Failure Survival)

| Test Layer | Scenarios | Passed | Pass Rate |
|------------|-----------|--------|-----------|
| Raw tool failures | 16 | 16 | 100% |
| Agent loop failures | 5 | 5 | 100% |
| Chained failure + recovery | 5 turns | 5/5 | 100% |
| **TOTAL** | **22** | **22** | **100%** |

The agent survived all 22 deliberate failure scenarios without crashing, including recovery after a mid-conversation tool failure.

---

### Latency Breakdown

| Category | Prompt A | Prompt B |
|----------|----------|----------|
| Fastest prompt | 1.226s (HP-01) | 1.232s (OOS-03) |
| Slowest prompt | 3.746s (AM-02) | 3.571s (OOS-01) |
| Mean (all 20) | 2.166s | 2.149s |
| Happy path mean | 1.856s | 1.838s |
| Ambiguous mean | 3.413s | 2.458s |
| Out-of-scope mean | 1.795s | 2.456s |

Prompt B was consistently faster across all categories, with the largest difference in ambiguous prompts (nearly 1 second faster).

---

### Tool Usage Distribution (Prompt A)

| Tool | Times Called | Success Rate |
|------|--------------|--------------|
| calculator | 5 | 100% |
| note_store | 5 | 100% |
| fact_lookup | 5 | 100% |
| abstain (no tool) | 5 | 100% |

Each tool was used exactly 5 times across the 20 prompts, and abstention occurred exactly on the 5 out-of-scope prompts.

---

### Deliberate Failures Tested (22 Total)

| Tool | Failure Type | Outcome |
|------|--------------|---------|
| Calculator | Division by zero | Caught by execute_tool |
| Calculator | Code injection attempt | Blocked by AST safety |
| Calculator | Empty expression | Caught by ValueError handler |
| Calculator | sqrt(-1) domain error | Caught by ValueError handler |
| Calculator | Malformed expression (2 ++ 2) | Calculator handled gracefully |
| Calculator | File access (open()) | Blocked by function blocking |
| NoteStore | Read nonexistent key | Returned not-found gracefully |
| NoteStore | Write with missing args | Caught by ValueError |
| NoteStore | Unknown action "explode" | Caught by ValueError |
| NoteStore | DB permission error | Caught by RuntimeError handler |
| NoteStore | Read with missing key | Caught by ValueError |
| FactLookup | simulate_timeout | Caught by ConnectionError |
| FactLookup | force_lookup_error | Caught by ConnectionError |
| FactLookup | Empty query | Returned not-found gracefully |
| FactLookup | Out-of-scope query | Returned not-found gracefully |
| Agent | Division by zero in loop | Responded gracefully |
| Agent | simulate_timeout in loop | Responded gracefully |
| Agent | Malformed expression | Abstained correctly |
| Agent | sqrt(-1) in loop | Responded gracefully |
| Agent | 5000-char input | Handled without error |
| Agent | Chain failure + recovery | All 5 turns passed |
| Unknown tool | nonexistent_tool | Returned "Unknown tool" error |

All 22 deliberate failure scenarios were handled without crashing the agent or leaking tracebacks to the user.

---

### Shipping Decision

| Prompt | Weighted Score | Decision |
|--------|----------------|----------|
| A (Conservative) | 0.7501 | Not shipped |
| B (Exploratory) | 0.7521 | Shipped |

Prompt B was selected for shipping because it matched Prompt A on accuracy (100%) while being marginally faster (2.149s vs 2.166s).

---

## One Line Summary

Perfect accuracy across 20 prompts, perfect survival across 22 failure scenarios, and Prompt B shipped for being 0.017 seconds faster.



# -> One Thing I Would Do Differently With Another Week

If I had another week, I would **add a fourth tool that is deliberately hard to distinguish from an existing tool** — specifically, a "WeatherTool" that overlaps with FactLookup.

---

### Why This Specifically

My current evaluation scored 100% across all prompts. That sounds good, but it actually tells me my test set wasn't hard enough. The agent never had to make a genuinely difficult trade-off.

The ambiguous prompts I designed (AM-01 through AM-05) were all resolved cleanly by the LLM. It always picked my preferred tool. That means the ambiguity was more theoretical than practical.

With another week, I would force real ambiguity.

---

### What I Would Build: WeatherTool

| Existing Tool | New Tool | Overlap Problem |
|---------------|----------|-----------------|
| FactLookup (encyclopaedic facts) | WeatherTool (current conditions) | User asks "What's the temperature in Paris?" — is that a fact (FactLookup) or live data (WeatherTool)? |

Both tools could plausibly answer. FactLookup knows "average temperature in Paris is 15°C". WeatherTool knows "it's 22°C right now". The agent would need to distinguish between *static fact* and *current condition*  a much harder decision than my current ambiguous cases.

---

### How I Would Redesign the Eval

| Current Ambiguous Prompt | Harder Version with WeatherTool |
|--------------------------|--------------------------------|
| "What is 20% of my budget?" (note_store vs calculator) | "What is the temperature in Paris?" (fact_lookup vs weather_tool) |
| "How far is London to Paris?" (fact_lookup vs calculator) | "Should I bring an umbrella tomorrow?" (weather_tool vs abstain) |
| "Boiling point in Fahrenheit?" (fact_lookup vs calculator) | "What's the record high temperature in Tokyo?" (fact_lookup vs weather_tool) |

The agent would have to reason about *time relevance*  does the user want a historical fact or current data? My current prompts don't test this at all.

---

### What Failure Modes I Would Expect

| Failure | Why It Would Happen |
|---------|---------------------|
| Agent calls WeatherTool for "average temperature" | LLM overgeneralizes "temperature" as live data |
| Agent calls FactLookup for "current weather" | LLM treats all factual-sounding queries as encyclopaedic |
| Agent calls both tools unnecessarily | Uncertainty leads to over-calling instead of abstaining |

With a WeatherTool, I would have exposed a deeper failure: **the agent cannot distinguish between static facts and dynamic data without explicit prompting.** That's a real architectural limitation of LLM tool selection, not just a wording issue.

---


### What I Would Not Change

| Current Decision | Would Keep | Why |
|------------------|------------|-----|
| Three-layer degradation tests | Yes | 22 failure scenarios proved crash-proof |
| SQLite for stateful storage | Yes | Real permission errors exposed real handling |
| Grok-3-mini over GPT-4 | Yes | Cost matters. 40 eval calls on GPT-4 would be expensive. |
| Two system prompts comparison | Yes | Prompt B won on speed. That's a real, measurable difference. |

---

### One Line Summary

With another week, I would add a WeatherTool that overlaps with FactLookup to create genuinely hard ambiguous cases because 100% accuracy means my test set wasn't challenging enough, not that my agent is perfect.

