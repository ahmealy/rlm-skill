---
name: rlm-pattern
description: >
  This skill should be used when the user asks to "reason iteratively over this",
  "use RLM pattern", "decompose this task with code", "analyze this large context",
  "use recursive language model reasoning", "iterate until you find the answer",
  "use REPL reasoning", "process this document iteratively", "apply RLM to this",
  or when the task involves context too large for a single prompt, requires
  multi-step code execution and reasoning, or needs sub-tasks delegated to sub-agents.
version: 0.1.0
---

# RLM Pattern — Iterative Reasoning Agent Skill

## Purpose

This skill implements the **Recursive Language Models (RLM)** reasoning pattern in agent
context. Instead of answering a query in one shot, run an iterative REPL-style loop:
write code → execute in terminal → read results → reason → repeat until a confident final
answer is produced. No human intervention between iterations.

This approximates `rlm.completion(prompt)` from the RLM library — the main difference is
that the agent drives each iteration rather than automated infrastructure.

---

## When to Apply This Pattern

Apply the RLM pattern when:
- The context is too large to fit in a single prompt (> 50K chars)
- The task requires code computation + language reasoning in combination
- Sub-tasks need their own iterative reasoning loops (`rlm_query`)
- Multiple chunks of context need to be processed and aggregated
- A single LLM answer is provably insufficient for correctness
- The task is a pipeline: decompose → process → aggregate → answer

Do NOT apply this pattern for simple factual questions, single-file code edits, or
tasks answerable with one terminal run.

---

## Session Setup

At the start of every RLM-pattern session, declare the plan (keeps the agent on track):

```
TASK: <one-line restatement of what to produce>
MAX_ITERATIONS: 10
ITERATIONS_USED: 0
```

Stop decomposing and synthesize when `ITERATIONS_USED >= MAX_ITERATIONS - 2`.

---

## Core Primitives

The RLM library provides named primitives. Each has a **native agent equivalent** (preferred)
and a **file-based fallback** (use if native fails or isn't available):

| RLM Primitive | Native (preferred) | File fallback |
|---|---|---|
| `context` variable | `state["context"]` in `_rlm_state.json` | Same — no alternative |
| `llm_query(prompt)` | Print data to stdout; agent reads and reasons naturally | Write `_rlm_query.json` → agent writes `_rlm_response.json` |
| `llm_query_batched(prompts)` | Write `_rlm_batch_query.json`; agent answers all and writes `_rlm_batch_response.json` | Print all chunks to stdout (only for tiny batches where structure doesn't matter) |
| `rlm_query(prompt)` | Spawn native subagent with prompt; get result back directly | Write `_subtask_N_input.json` → run `_subtask_N.py` → read `_subtask_N_result.json` |
| `rlm_query_batched(prompts)` | Spawn ≤5 subagents in parallel; collect all results | Write N separate `_subtask_N_input.json` files; run `_subtask_N.py` for each; collect results |
| `FINAL_VAR(var)` | `print(f"FINAL_ANSWER: {state['var']}")` | Write `_final_answer.json` |
| `FINAL(text)` | `print(f"FINAL_ANSWER: {text}")` | Same |
| `SHOW_VARS()` | Print all keys + types from `_rlm_state.json` | Same — no alternative |
| `max_iterations` | `MAX_ITER = 10` tracked in `state["iteration"]` | Same |

Full code patterns for each primitive: see **`references/primitives.md`**.

---

## Core Workflow (5 Steps)

### Step 1: Load Context and Initialize State

On iteration 0, load the context into `_rlm_state.json`:

```python
import json, os

STATE_FILE = "_rlm_state.json"
state = json.load(open(STATE_FILE)) if os.path.exists(STATE_FILE) else {}

if "context" not in state:
    context = open("input.txt").read()  # or inline, or JSON
    state.update({
        "context": context,
        "query": "the user's original question",
        "iteration": 0,
        "buffer": [],
        "error_count": 0,
    })
    print(f"[init] context: {type(context).__name__}, {len(str(context))} chars")

json.dump(state, open(STATE_FILE, "w"), indent=2, default=str)
```

Inspect the context — print a preview, check its type and length, decide chunking strategy.

### Step 2: Plan the Approach

Before writing code, reason about the strategy:
- Is this context a string, list, or dict?
- Does it fit in a single agent call (< 50K chars) or need chunking?
- Are sub-tasks independent (use `llm_query_batched`) or need sequential reasoning?
- Does any sub-task need its own iteration loop (use `rlm_query`)?

Commit to a strategy before starting iteration 1.

### Step 3: Execute One Iteration

Each iteration runs a Python script in terminal. Use the **standard header**:

```python
import json, os

STATE_FILE = "_rlm_state.json"
MAX_ITER = 10
state = json.load(open(STATE_FILE))
state["iteration"] = state.get("iteration", 0) + 1
print(f"[iter {state['iteration']}/{MAX_ITER}]")

# --- work for this iteration ---
# Compute, parse, transform in Python. No need for llm_query on deterministic steps.
# Use llm_query (file handoff) only for semantic/language tasks.

# Save state
json.dump(state, open(STATE_FILE, "w"), indent=2, default=str)
```

### Step 4: Read Results and Reason

After each terminal run, read the stdout output. Decide:
- Is the result sufficient to answer the query? → Signal FINAL_ANSWER.
- Does a chunk need language understanding? → Reason about it directly (agent reads stdout). If the platform's native reasoning is unavailable, fall back to writing `_rlm_query.json`.
- Does a sub-task need its own iteration loop? → Spawn a native subagent. If subagents are unavailable, fall back to `_subtask_N.py` file approach.
- Are there more chunks to process? → Continue to next iteration.

### Step 5: Signal Completion

When the answer is ready, signal it exactly once — never continue after this:

```python
print(f"FINAL_ANSWER: {state['final_answer']}")
```

Or for structured answers:
```python
import json
json.dump({"final_answer": state["final_answer"]}, open("_final_answer.json", "w"), indent=2)
print("FINAL_ANSWER: written to _final_answer.json")
```

### Step 6: Clean Up

Optionally remove working files after signaling FINAL_ANSWER:

```python
import os, glob
for f in glob.glob("_rlm_*.json") + glob.glob("_subtask_*.json"):
    os.remove(f)
print("[cleanup] working files removed")
```

---

## Chunking Decision

Select chunking strategy based on context size:

| Context | Strategy |
|---|---|
| < 50K chars | No chunking — pass directly to agent reasoning |
| 50K–500K chars | Split into 50K chunks, batch query all chunks, aggregate |
| > 500K chars | Split into 100K chunks, process in groups of 5, aggregate in stages |
| `List[str]` / `List[dict]` | Each item = one unit; batch up to 10 per query |

Detailed chunking code: see **`references/reasoning-pattern.md`** → "Chunking Strategies".

---

## Sub-Query Decision: llm_query vs rlm_query

**llm_query** — for semantic tasks the agent can answer from stdout alone:
- Summarization of a chunk
- Extracting a fact from a passage
- Classification or sentiment of a piece of text
- Simple Q&A over a section

**Native approach:** print the data to stdout; the agent reads and reasons about it between terminal runs. No extra files needed.

**File fallback** (if native reasoning doesn't work): write `_rlm_query.json`, read `_rlm_response.json`. See `references/primitives.md` → "llm_query File Fallback".

---

**rlm_query** — for sub-tasks that need their own multi-step iteration loop:
- A sub-problem requiring code execution + reasoning over multiple steps
- An analysis where a single agent response is provably insufficient
- A well-scoped subtask with a clear expected output format

**Native approach:** spawn a subagent with the sub-task prompt; include budget (`MAX_ITERATIONS - ITERATIONS_USED`) and expected output format in the prompt.

**File fallback** (if subagents unavailable): write `_subtask_N_input.json`, run `_subtask_N.py`, read `_subtask_N_result.json`. See `references/primitives.md` → "rlm_query File Fallback".

Do NOT spawn a subagent for:
- Simple lookups or single terminal commands
- Tasks that need current in-memory state (use `_rlm_state.json` instead)
- More than 5 subagents at once (batch via `llm_query_batched` instead)

Default to `llm_query`. Escalate to `rlm_query` only when genuinely multi-step.

---

## Iteration Limits and Error Handling

Track iteration count in every script:

```python
if state["iteration"] >= MAX_ITER:
    best = state.get("buffer", [])
    fallback = best[-1] if best else "Reached iteration limit without answer"
    print(f"FINAL_ANSWER: {fallback}")
```

Track consecutive errors:

```python
try:
    # work
    state["error_count"] = 0
except Exception as e:
    state["error_count"] = state.get("error_count", 0) + 1
    print(f"[error {state['error_count']}] {e}")
    if state["error_count"] >= 3:
        print(f"FINAL_ANSWER: error threshold — {state.get('buffer', ['no result'])[-1]}")
```

Full error and limit patterns: see **`references/reasoning-pattern.md`** → "Error Tracking".

---

## Key Rules

1. **Declare plan at session start** — state TASK, MAX_ITERATIONS, ITERATIONS_USED
2. **Native first, file fallback second** — use stdout reasoning before file handoff; use subagents before sub-scripts
3. **Save state after every script** — always `json.dump(state, open(STATE_FILE, "w"), ...)`
4. **Use Python for determinism** — math, parsing, sorting → Python; semantics → agent reasoning
5. **One FINAL_ANSWER only** — print it once, then stop
6. **Inspect context first** — always print a preview before processing
7. **Accumulate into buffer** — `state["buffer"].append(result)` after each meaningful step
8. **Stop decomposing at `MAX_ITER - 2`** — synthesize with what exists rather than spawning more work
9. **`llm_query` before `rlm_query`** — default to the lighter pattern; escalate only when sub-task is genuinely multi-step

---

## Anti-Patterns to Avoid

- **Don't spawn a subagent for a single terminal command** — run it directly
- **Don't use file handoff when stdout reasoning works** — unnecessary complexity
- **Don't use stdout for `llm_query_batched`** — structured N-answer batches need `_rlm_batch_query.json`
- **Don't loop past `MAX_ITER`** — synthesize early with `buffer[-1]` if needed
- **Don't carry large data blobs between iterations in memory** — write to `_rlm_state.json`
- **Don't signal FINAL_ANSWER before `state["final_answer"]` exists** — assign in script first
- **Don't spawn >5 subagents at once via `rlm_query_batched`** — use `llm_query_batched` instead
- **Don't leave working files behind** — clean up `_rlm_*.json` and `_subtask_*.json` after completion

---

## Additional Resources

### Reference Files

Consult these for detailed patterns and complete code snippets:

- **`references/primitives.md`** — Full code for every primitive (llm_query, rlm_query,
  FINAL_VAR, SHOW_VARS, state save/load, standard header/footer)
- **`references/reasoning-pattern.md`** — Chunking strategies, error tracking, depth
  simulation, iteration budget, cleanup, system prompt origin

### Source

This skill is derived from the **[rlm](https://github.com/alexzhang13/rlm)** library
(package: `rlms`, arXiv: 2512.24601). If working inside that repo:

- `rlm/utils/prompts.py` → `RLM_SYSTEM_PROMPT` — original system prompt this skill adapts
- `rlm/environments/local_repl.py` — canonical primitive implementations (llm_query, rlm_query, FINAL_VAR, SHOW_VARS)
- `rlm/core/rlm.py` → `completion()` — the full iterative orchestration loop

If working in a standalone repo that copied this skill, consult the upstream library for context on any primitive or pattern not covered in the reference files above.

**Custom tools:** The RLM library supports `custom_tools` — a dict of callable functions injected into the REPL globals alongside `llm_query` etc. In agent mode, define helper functions at the top of each script instead. If a function is reused across iterations, persist its definition string in `state["helpers"]` and `exec()` it at script start.
