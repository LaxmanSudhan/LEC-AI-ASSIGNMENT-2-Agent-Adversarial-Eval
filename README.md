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


