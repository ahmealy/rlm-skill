# RLM Reasoning Pattern — Detailed Reference

This reference covers the full Recursive Language Models (RLM) reasoning pattern as adapted for
agent use (Copilot / Claude Code). It documents chunking strategies, sub-query patterns,
state persistence, completion signaling, depth simulation, and error tracking.

---

## Core Mental Model

The RLM pattern replaces a single LLM one-shot answer with an **iterative REPL loop**:

```
while not done and iteration < max_iterations:
    1. Inspect context / state
    2. Write and run code (in terminal)
    3. Read stdout / results
    4. Reason about what to do next
    5. Either: continue iterating OR signal FINAL_ANSWER
```

In `rlm.completion()` this loop is fully automated. In agent mode the **agent drives each
iteration** — reading output, deciding the next step, writing the next code block, running it.
The human does not intervene. The agent continues until it signals completion.

---

## State Persistence Between Terminal Runs

Terminal runs are isolated — variables don't survive between separate `python` invocations.
Use `_rlm_state.json` as the persistent namespace:

```python
# At start of every script: load existing state
import json, os

STATE_FILE = "_rlm_state.json"
state = json.load(open(STATE_FILE)) if os.path.exists(STATE_FILE) else {}

# ... do work ...

# At end of every script: save state back
state["my_variable"] = result
json.dump(state, open(STATE_FILE, "w"), indent=2, default=str)
print(f"[state] saved keys: {list(state.keys())}")
```

**Conventions:**
- `state["context"]` — loaded input context (string, dict, or list)
- `state["iteration"]` — current iteration number (increment each run)
- `state["buffer"]` — accumulated intermediate results
- `state["final_answer"]` — set this when done
- `state["error_count"]` — track consecutive errors

**SHOW_VARS equivalent:**
```python
import json
state = json.load(open("_rlm_state.json"))
for k, v in state.items():
    val_preview = str(v)[:100] if not isinstance(v, (int, float, bool)) else v
    print(f"  {k}: {type(v).__name__} = {val_preview}")
```

---

## Context Loading Strategies

### Strategy 1: Small context (< 50K chars) — load directly

```python
# Load context into state on iteration 0
with open("input.txt") as f:
    context = f.read()
state["context"] = context
state["context_length"] = len(context)
print(f"[context] loaded {len(context)} chars")
```

### Strategy 2: Large context — chunked approach

```python
# Determine chunk strategy
context = state["context"]  # already loaded
chunk_size = 50_000  # chars per chunk
chunks = [context[i:i+chunk_size] for i in range(0, len(context), chunk_size)]
state["chunks"] = chunks
state["n_chunks"] = len(chunks)
state["current_chunk"] = 0
print(f"[chunking] {len(chunks)} chunks of ~{chunk_size} chars each")
```

### Strategy 3: Structured context (dict/list)

```python
# Context is a list of documents
import json
with open("context.json") as f:
    docs = json.load(f)
state["docs"] = docs
state["n_docs"] = len(docs)
print(f"[context] {len(docs)} documents loaded")
for i, d in enumerate(docs[:3]):
    preview = str(d)[:200]
    print(f"  doc[{i}]: {preview}...")
```

---

## llm_query() — Native vs File Fallback

### Native approach (preferred)

In agent mode, the agent IS the LLM. After every terminal run, the agent reads stdout and
reasons about it. That reasoning **is** `llm_query()`. No files needed.

Just print what needs language understanding:
```python
# Print the chunk — agent reads and reasons between terminal runs
print(f"[llm_query] passage to analyze:")
print(chunk[:5000])
print("[end passage]")
# In the NEXT script, the agent incorporates its conclusion into state directly
```

This covers: summarization, extraction, classification, Q&A over a passage.

### File Fallback

Use when the native approach isn't working (stdout reasoning not being picked up,
or the platform doesn't persist reasoning between runs):

**Script writes query:**
```python
import json

query = {
    "prompt": f"Summarize this passage and extract key facts:\n\n{chunk}",
    "context_hint": "passage from document chunk 3"
}
json.dump(query, open("_rlm_query.json", "w"), indent=2)
print("[llm_query fallback] query written to _rlm_query.json — agent will respond next")
```

**Agent response format** (agent writes this before next script run):
```json
{ "answer": "<the agent's response here>" }
```

**Next script reads response:**
```python
import json, os
if os.path.exists("_rlm_response.json"):
    resp = json.load(open("_rlm_response.json"))
    answer = resp["answer"]
    state["buffer"].append(answer)
    os.remove("_rlm_response.json")
    print(f"[llm_response] {answer[:200]}")
json.dump(state, open("_rlm_state.json", "w"), indent=2, default=str)
```

---

## llm_query_batched() — Parallel Sub-Queries

For multiple independent sub-queries to the agent, write them all to a batch file then have
the agent answer each one in a single pass:

```python
import json

# Build all queries upfront
chunks = state["chunks"]
queries = [
    {"id": i, "prompt": f"What are the key facts in this passage? Passage:\n\n{chunk}"}
    for i, chunk in enumerate(chunks)
]
json.dump(queries, open("_rlm_batch_query.json", "w"), indent=2)
print(f"[batch] {len(queries)} queries written — agent will answer all")
```

After agent answers (writes `_rlm_batch_response.json`):
```python
import json
responses = json.load(open("_rlm_batch_response.json"))
# responses is a list of {"id": N, "answer": "..."}
answers = {r["id"]: r["answer"] for r in responses}
state["chunk_answers"] = answers
print(f"[batch] received {len(answers)} answers")
```

---

## rlm_query() — Sub-Agent (Native vs File Fallback)

When a subtask requires its own multi-step iterative reasoning:

### Native approach (preferred)

Spawn a native subagent with the sub-task prompt. Always include:
- The specific sub-task (not the full parent context)
- Budget: `MAX_ITERATIONS - ITERATIONS_USED` remaining
- Expected output format

```
Spawn subagent:
  Sub-task: Analyze this dataset and determine the trend. Single-word answer: up, down, or stable.
  Data: <paste state["relevant_data_slice"] here>
  Budget: 5 iterations remaining
  Expected output: one word (up/down/stable) + one-sentence reasoning
```

Read result and branch in Python:
```python
trend = "up"  # agent fills in from subagent result
state["trend"] = trend
if "up" in trend.lower():
    recommendation = "Consider increasing exposure."
elif "down" in trend.lower():
    recommendation = "Consider hedging."
else:
    recommendation = "Hold position."
state["recommendation"] = recommendation
print(f"[rlm_query result] trend={trend}, rec={recommendation}")
json.dump(state, open("_rlm_state.json", "w"), indent=2, default=str)
```

Do NOT spawn subagents for single commands, tasks needing current in-memory variables,
or more than 5 independent subtasks at once.

### File Fallback

Use when native subagent spawning is unavailable or fails:

1. Write `_subtask_N_input.json` with the sub-problem, context slice, and `max_iter`
2. Agent writes `_subtask_N.py` as an independent script with its own `_subtask_N_state.json`
3. Sub-script iterates independently and writes final answer to `_subtask_N_result.json`
4. Parent reads result and cleans up

Full fallback code: see `references/primitives.md` → "rlm_query File Fallback".

**Depth control:** Sub-scripts should use native stdout reasoning rather than spawning
further sub-scripts. At depth 2, fall back to direct reasoning.

---

## Chunking Strategies

### Strategy selection guide

| Context size | Type | Strategy |
|---|---|---|
| < 50K chars | Any | Load directly, no chunking |
| 50K–500K chars | String | Fixed chunks of 50K (up to 10 queries) |
| 500K–5M chars | String | Fixed chunks of 100K, batched |
| Any size | List[str] | Each item is one unit; batch up to 10 per query |
| Any size | List[dict] | Serialize each item; batch based on total size |

### Fixed-size chunking
```python
chunk_size = 50_000
chunks = [context[i:i+chunk_size] for i in range(0, len(context), chunk_size)]
```

### Semantic chunking (paragraph-aware)
```python
paragraphs = context.split("\n\n")
chunks, current, current_len = [], [], 0
for p in paragraphs:
    if current_len + len(p) > 50_000 and current:
        chunks.append("\n\n".join(current))
        current, current_len = [], 0
    current.append(p)
    current_len += len(p)
if current:
    chunks.append("\n\n".join(current))
```

### Aggregation after chunking
```python
# After getting answers per chunk, aggregate
answers = state["chunk_answers"]  # dict: {chunk_id: answer_str}
combined = "\n\n---\n\n".join(
    f"[Chunk {i}]\n{answers[i]}" for i in sorted(answers)
)
# Write aggregation query for agent to answer
json.dump(
    {"prompt": f"Synthesize these per-chunk answers into a final answer for: {state['query']}\n\n{combined}"},
    open("_rlm_query.json", "w")
)
```

---

## Completion Signaling

Signal task completion in exactly one of two ways:

### Option 1: Direct print (preferred for simple answers)
```python
print(f"FINAL_ANSWER: {final_answer}")
```

### Option 2: File (preferred for long/structured answers)
```python
import json
json.dump({"final_answer": final_answer, "iteration": state["iteration"]},
          open("_final_answer.json", "w"), indent=2)
print("FINAL_ANSWER: written to _final_answer.json")
```

**After signaling:** Stop writing new scripts. The agent reads the final answer and delivers
it to the user.

**Never:** Signal FINAL_ANSWER before exhausting all iterations or before actually having a
complete answer. If uncertain, run one more iteration to verify.

---

## Error Tracking and Recovery

Track consecutive errors to avoid infinite bad loops:

```python
MAX_CONSECUTIVE_ERRORS = 3

try:
    # ... main logic ...
    state["error_count"] = 0  # reset on success
except Exception as e:
    state["error_count"] = state.get("error_count", 0) + 1
    state["last_error"] = str(e)
    print(f"[error] {e} (count={state['error_count']})")
    if state["error_count"] >= MAX_CONSECUTIVE_ERRORS:
        # Fall back: use whatever partial answer exists
        partial = state.get("buffer", ["No result yet"])
        state["final_answer"] = f"[partial] {partial[-1] if partial else 'error'}"
        print(f"FINAL_ANSWER: error threshold reached, using partial: {state['final_answer']}")
```

---

## Iteration Budget

Default max iterations: **10**. Track explicitly:

```python
MAX_ITER = 10
state["iteration"] = state.get("iteration", 0) + 1

if state["iteration"] >= MAX_ITER:
    # Force completion — use best available answer
    best = state.get("buffer", ["No answer found"])
    print(f"FINAL_ANSWER: {best[-1] if best else 'max iterations reached without answer'}")
    state["final_answer"] = best[-1] if best else "Reached iteration limit"
```

---

## Truncated Output Warning

The RLM library explicitly warns: **REPL stdout is truncated** — long outputs are cut off.
The same applies in agent mode: terminal output is truncated if too long.

Mitigation strategies:
- Never print an entire large variable directly — print a preview only:
  ```python
  print(f"[context preview] {str(context)[:500]}...")
  print(f"[context total] {len(str(context))} chars")
  ```
- Use `llm_query` / agent reasoning on variables rather than printing them raw:
  ```python
  # BAD: print(big_text)  — will truncate
  # GOOD: use llm_query to extract key info from the text
  answer = llm_query(f"Extract key facts from:\n\n{big_text}")
  print(f"[extracted] {answer}")
  ```
- Write large intermediate outputs to `_rlm_state.json` and read selectively:
  ```python
  state["large_output"] = result
  json.dump(state, open(STATE_FILE, "w"), indent=2, default=str)
  print(f"[state] saved large_output ({len(str(result))} chars)")
  ```
- When debugging state, use SHOW_VARS to read key names first, then load only what's needed.

---

## Computation → llm_query Chaining

The REPL is for deterministic computation. Use Python for math, parsing, sorting, distances,
formulas. Then pass the *computed numbers* to the agent (via stdout reasoning or `llm_query`)
for interpretation or final phrasing.

Example: physics problem (helical motion entry angle)
```python
import math, json

# Extract numerical values from context (already parsed in earlier iteration)
state = json.load(open("_rlm_state.json"))
B = state["B"]      # magnetic field strength
m = state["m"]      # electron mass
q = state["q"]      # charge
pitch = state["pitch"]
R = state["R"]      # radius

# Compute deterministically in Python
v_parallel = pitch * (q * B) / (2 * math.pi * m)
v_perp = R * (q * B) / m
theta_deg = math.degrees(math.atan2(v_perp, v_parallel))

print(f"[computation] v_parallel={v_parallel:.4f}, v_perp={v_perp:.4f}")
print(f"[computation] entry angle={theta_deg:.2f} degrees")

# Agent reads the numbers above and phrases the final answer
# — or use llm_query file fallback with the computed value:
state["computed_angle_deg"] = theta_deg
json.dump(state, open("_rlm_state.json", "w"), indent=2, default=str)
```

General pattern:
1. Python computes intermediate quantities (no LLM involved)
2. Print the numbers to stdout
3. Agent reads the numbers and reasons about interpretation / final answer
4. Store computed values in state; only call `llm_query` for the language layer

This avoids asking the agent to do arithmetic — let Python handle numbers.

---

## Clean-Up

After signaling FINAL_ANSWER, optionally clean up working files:

```python
import os, glob
for f in glob.glob("_rlm_*.json") + glob.glob("_subtask_*.json"):
    os.remove(f)
print("[cleanup] working files removed")
```
