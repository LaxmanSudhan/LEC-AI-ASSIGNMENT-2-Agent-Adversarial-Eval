# LEC-AI-ASSIGNMENT-2-Agent-Adversarial-Eval

## Part 1: Tool Design & Justification

### Three Tools

| Tool | Type | Purpose |
|------|------|---------|
| **Calculator** | Stateless | Math operations |
| **NoteStore** | Stateful (SQLite) | Persistent memory across turns |
| **FactLookup** | Stateless | Static knowledge retrieval |

### Why These Three?

Not three search variants. Each tool serves a **distinct domain**:
- **Calculator** → computation only
- **NoteStore** → user-provided information (write/read/delete/list)
- **FactLookup** → external world facts

The ambiguity comes from **overlap** (e.g., "remember X" vs "know X"), not from similarity.

### Deliberate Failures Built In

| Tool | Failure Trigger | Exception |
|------|----------------|-----------|
| Calculator | Division by zero | `ZeroDivisionError` |
| NoteStore | Invalid DB path | `RuntimeError` |
| FactLookup | `simulate_timeout` keyword | `ConnectionError` |

All exceptions propagate to agent for graceful degradation testing.

---

## Experiment Design Decisions

### Decision 1: Exceptions, not error dicts
Forces agent to use real `try/except`. Returning `{"error": ...}` allows ignoring.

### Decision 2: SQLite persistence, not in-memory
Exposes real failures (permissions, corruption). Matches real-world agents.

### Decision 3: FactLookup returns `None` on not-found
Better to fail honestly than hallucinate. Assignment punishes hallucination.

---
