# RLM Primitives — Agent Adaptation Reference

This reference documents the exact mapping between RLM library primitives and their agent
(Copilot / Claude Code) equivalents. Use these patterns as building blocks.

---

## Primitive Mapping Table

| RLM Library | Native (preferred) | File fallback |
|---|---|---|
| `context` variable | `state["context"]` in `_rlm_state.json` | Same — no alternative |
| `llm_query(prompt)` | Print data to stdout; agent reads naturally | `_rlm_query.json` / `_rlm_response.json` |
| `llm_query_batched(prompts)` | `_rlm_batch_query.json` → agent answers all → `_rlm_batch_response.json` | Print all chunks to stdout (only for tiny, unstructured batches) |
| `rlm_query(prompt)` | Spawn native subagent with prompt | `_subtask_N_input.json` → `_subtask_N.py` → `_subtask_N_result.json` |
| `rlm_query_batched(prompts)` | Spawn ≤5 subagents in parallel; collect all results | Write N separate `_subtask_N_input.json`; run each `_subtask_N.py`; collect all results |
| `FINAL_VAR(var_name)` | `print(f"FINAL_ANSWER: {state['var_name']}")` | Write `_final_answer.json` |
| `FINAL(answer)` | `print(f"FINAL_ANSWER: {answer}")` | Same |
| `SHOW_VARS()` | Print all keys + type + preview from `_rlm_state.json` | Same |
| `max_iterations` | `MAX_ITER = 10` constant + `state["iteration"]` | Same |
| `consecutive_errors` | `state["error_count"]` | Same |
| `partial_answer` | `state.get("buffer", [])[-1]` | Same |

---

## Context Primitive

### Loading context on iteration 0

```python
import json, os

STATE_FILE = "_rlm_state.json"
state = json.load(open(STATE_FILE)) if os.path.exists(STATE_FILE) else {}

# On first iteration, load context
if "context" not in state:
    # Option A: inline string
    context = """<paste context here>"""

    # Option B: from file
    # context = open("context.txt").read()

    # Option C: from JSON
    # context = json.load(open("context.json"))

    state["context"] = context
    state["iteration"] = 0
    state["buffer"] = []
    state["error_count"] = 0
    print(f"[init] context loaded: {type(context).__name__}, length={len(str(context))}")

json.dump(state, open(STATE_FILE, "w"), indent=2, default=str)
```

---

## llm_query Primitive

### Native approach (preferred)

Print data to stdout — the agent reads it and reasons about it naturally between terminal runs.
This IS `llm_query()`. No files needed.

```python
# Just print what the agent needs to reason about
print(f"[llm_query] Summarize this passage and extract key facts:")
print(chunk)
print("[llm_query] agent: please reason about the above before the next script run")
```

The agent reads stdout, reasons about it, then writes the next script incorporating its conclusions.
Use `state["buffer"]` to carry those conclusions forward:

```python
# In the next script, the agent writes its conclusion directly:
state["buffer"].append("The main claim is that ...")  # agent fills this in
json.dump(state, open(STATE_FILE, "w"), indent=2, default=str)
```

### llm_query File Fallback

Use when the native approach isn't working (e.g. stdout reasoning not being picked up,
or the platform doesn't support inter-turn reasoning):

**Script writes query:**
```python
import json

query = {
    "prompt": f"Extract the key claim from this passage:\n\n{chunk}",
    "hint": "one sentence answer"
}
json.dump(query, open("_rlm_query.json", "w"), indent=2)
print("[llm_query fallback] query written to _rlm_query.json — agent will respond")
# Script stops here. Agent reads _rlm_query.json and writes _rlm_response.json.
```

**Agent response format** (agent writes this before next script run):
```json
{
  "answer": "The main claim is that ..."
}
```

**Next script reads response:**
```python
import json, os

if os.path.exists("_rlm_response.json"):
    resp = json.load(open("_rlm_response.json"))
    answer = resp["answer"]
    os.remove("_rlm_response.json")
    state["buffer"].append(answer)
    print(f"[llm_query result] {answer[:300]}")
json.dump(state, open("_rlm_state.json", "w"), indent=2, default=str)
```

---

## llm_query_batched Primitive

Use for N independent semantic sub-queries in one structured pass. Unlike `llm_query`,
the file approach is **primary** here — stdout batch output is hard to parse reliably for
N structured answers.

### Primary approach: batch query file

**Script writes batch:**
```python
import json

chunks = state["chunks"]
queries = [
    {
        "id": i,
        "prompt": f"Summarize chunk {i} and identify the main topic:\n\n{chunk}"
    }
    for i, chunk in enumerate(chunks)
]
json.dump(queries, open("_rlm_batch_query.json", "w"), indent=2)
print(f"[llm_query_batched] {len(queries)} queries written → agent will answer all")
```

**Agent response format** (agent writes `_rlm_batch_response.json`):
```json
[
  {"id": 0, "answer": "..."},
  {"id": 1, "answer": "..."},
  {"id": 2, "answer": "..."}
]
```

**Next script reads batch response:**
```python
import json, os

responses = json.load(open("_rlm_batch_response.json"))
answers = {r["id"]: r["answer"] for r in responses}
os.remove("_rlm_batch_response.json")

state["chunk_answers"] = answers
print(f"[batch result] {len(answers)} answers received")
for i, ans in sorted(answers.items()):
    print(f"  [{i}] {ans[:100]}")

json.dump(state, open("_rlm_state.json", "w"), indent=2, default=str)
```

---

## rlm_query Primitive (Sub-Agent)

### Native approach (preferred)

Spawn a native subagent with the sub-task prompt. Include budget and expected output format:

```
Spawn subagent:
  Sub-task: Analyze this dataset and determine the trend. Single-word answer: up, down, or stable.
  Data: <paste relevant slice from state>
  Budget: 5 iterations
  Expected output: one word followed by one-sentence reasoning
```

Read the subagent's result and incorporate it into state:

```python
# After subagent returns:
trend = "up"  # agent fills in from subagent result
state["trend"] = trend
if "up" in trend.lower():
    state["recommendation"] = "Consider increasing exposure."
elif "down" in trend.lower():
    state["recommendation"] = "Consider hedging."
else:
    state["recommendation"] = "Hold position."
print(f"[rlm_query result] trend={trend}")
json.dump(state, open("_rlm_state.json", "w"), indent=2, default=str)
```

### rlm_query File Fallback

Use when native subagent spawning is unavailable or fails:

**Parent writes sub-task input:**
```python
import json

subtask = {
    "subtask_id": "subtask_0",
    "prompt": state["sub_problem"],
    "context": state.get("relevant_context", ""),
    "max_iter": 5
}
json.dump(subtask, open("_subtask_0_input.json", "w"), indent=2)
print("[rlm_query fallback] subtask_0 defined — agent writes and runs _subtask_0.py")
```

**Sub-script template** (agent writes this as `_subtask_0.py`):
```python
import json, os

INPUT_FILE = "_subtask_0_input.json"
STATE_FILE = "_subtask_0_state.json"
RESULT_FILE = "_subtask_0_result.json"

inp = json.load(open(INPUT_FILE))
state = json.load(open(STATE_FILE)) if os.path.exists(STATE_FILE) else {
    "iteration": 0, "buffer": [], "error_count": 0,
    "context": inp.get("context", ""),
    "prompt": inp["prompt"]
}

MAX_ITER = inp.get("max_iter", 5)
state["iteration"] += 1
print(f"[subtask_0] iteration {state['iteration']}/{MAX_ITER}")

# --- sub-task work here ---

# Signal completion
if done or state["iteration"] >= MAX_ITER:
    json.dump(
        {"answer": final_answer, "iterations_used": state["iteration"]},
        open(RESULT_FILE, "w"), indent=2
    )
    print(f"[subtask_0] DONE: {final_answer[:200]}")
else:
    json.dump(state, open(STATE_FILE, "w"), indent=2, default=str)
    print(f"[subtask_0] continuing — run _subtask_0.py again")
```

**Parent reads sub-result:**
```python
import json, os

result = json.load(open("_subtask_0_result.json"))
answer = result["answer"]

for f in ["_subtask_0_input.json", "_subtask_0_state.json", "_subtask_0_result.json"]:
    if os.path.exists(f):
        os.remove(f)

state["sub_result"] = answer
print(f"[rlm_query result] {answer[:300]}")
json.dump(state, open("_rlm_state.json", "w"), indent=2, default=str)
```

---

## rlm_query_batched Primitive (Parallel Sub-Agents)

Use when multiple independent sub-tasks each need their own multi-step iteration loop.

### Native approach (preferred)

Spawn N subagents in parallel. Give each its own prompt, data slice, budget, and output format.
Wait for all to complete, then aggregate in a script:

```python
# After all N subagents return:
import json

results = [
    "subagent_0_result",  # agent fills each in from subagent output
    "subagent_1_result",
    "subagent_2_result",
]
state["sub_results"] = results
aggregated = "\n\n".join(f"[Subtask {i}]\n{r}" for i, r in enumerate(results))
state["aggregated"] = aggregated
print(f"[rlm_query_batched] {len(results)} results collected")
json.dump(state, open("_rlm_state.json", "w"), indent=2, default=str)
```

### File fallback

Write one `_subtask_N_input.json` per sub-task, run each `_subtask_N.py` independently,
then collect all `_subtask_N_result.json` files:

```python
import json

sub_tasks = state["sub_problems"]  # list of independent problems
for i, problem in enumerate(sub_tasks):
    subtask = {
        "subtask_id": f"subtask_{i}",
        "prompt": problem,
        "context": state.get("relevant_context", ""),
        "max_iter": 5
    }
    json.dump(subtask, open(f"_subtask_{i}_input.json", "w"), indent=2)
    print(f"[rlm_query_batched fallback] subtask_{i} written")
print(f"Agent: run _subtask_0.py through _subtask_{len(sub_tasks)-1}.py, then collect results")
```

After all sub-scripts complete, collect results:
```python
import json, os

results = []
for i in range(len(state["sub_problems"])):
    res = json.load(open(f"_subtask_{i}_result.json"))
    results.append(res["answer"])
    for f in [f"_subtask_{i}_input.json", f"_subtask_{i}_state.json", f"_subtask_{i}_result.json"]:
        if os.path.exists(f): os.remove(f)

state["sub_results"] = results
print(f"[rlm_query_batched] {len(results)} results collected")
json.dump(state, open("_rlm_state.json", "w"), indent=2, default=str)
```

---

## FINAL_VAR / FINAL Primitives

### Signaling completion with a variable

```python
import json

# Ensure the variable is in state
final_answer = state["final_answer"]  # must exist before calling

# Option A: print directly
print(f"FINAL_ANSWER: {final_answer}")

# Option B: write to file (for long/structured answers)
json.dump(
    {"final_answer": final_answer, "iteration": state["iteration"]},
    open("_final_answer.json", "w"),
    indent=2
)
print("FINAL_ANSWER: written to _final_answer.json")
```

**Rule:** Only signal `FINAL_ANSWER:` once. After printing it, do not run more scripts.

**WARNING — common mistake (mirrors RLM library warning):**
- WRONG: Print `FINAL_ANSWER:` before assigning `state["final_answer"]`
- CORRECT: Assign and save the variable in a script first, then signal in the SAME or NEXT script

---

## SHOW_VARS Primitive

### Inspect current namespace

```python
import json, os

STATE_FILE = "_rlm_state.json"
if not os.path.exists(STATE_FILE):
    print("[SHOW_VARS] no state file yet")
else:
    state = json.load(open(STATE_FILE))
    print(f"[SHOW_VARS] {len(state)} variables:")
    for k, v in state.items():
        if isinstance(v, (int, float, bool, type(None))):
            preview = str(v)
        elif isinstance(v, str):
            preview = repr(v[:80]) + ("..." if len(v) > 80 else "")
        elif isinstance(v, list):
            preview = f"list[{len(v)}], first={str(v[0])[:60] if v else 'empty'}"
        elif isinstance(v, dict):
            preview = f"dict[{len(v)} keys]: {list(v.keys())[:5]}"
        else:
            preview = type(v).__name__
        print(f"  {k}: {preview}")
```

---

## Standard Script Header

Include this boilerplate at the top of every script in an RLM-pattern session:

```python
import json
import os

STATE_FILE = "_rlm_state.json"
MAX_ITER = 10

# Load state
state = json.load(open(STATE_FILE)) if os.path.exists(STATE_FILE) else {}
state["iteration"] = state.get("iteration", 0) + 1
print(f"[iter {state['iteration']}/{MAX_ITER}] starting")

# ... work ...

# Save state at end
json.dump(state, open(STATE_FILE, "w"), indent=2, default=str)
print(f"[iter {state['iteration']}] state saved, keys={list(state.keys())}")
```

---

## Standard Script Footer (with completion check)

```python
# At bottom of every script
if state.get("final_answer"):
    print(f"FINAL_ANSWER: {state['final_answer']}")
elif state["iteration"] >= MAX_ITER:
    best = state.get("buffer", [])
    fallback = best[-1] if best else "No answer found within iteration limit"
    state["final_answer"] = fallback
    json.dump(state, open(STATE_FILE, "w"), indent=2, default=str)
    print(f"FINAL_ANSWER: {fallback}")
else:
    json.dump(state, open(STATE_FILE, "w"), indent=2, default=str)
    print(f"[iter {state['iteration']}] complete — ready for next iteration")
```

---

## File Naming Conventions

| File | Purpose |
|---|---|
| `_rlm_state.json` | Main persistent namespace (equivalent to REPL locals) |
| `_rlm_query.json` | Single sub-query to agent |
| `_rlm_response.json` | Agent's answer to single sub-query |
| `_rlm_batch_query.json` | Batch sub-queries to agent |
| `_rlm_batch_response.json` | Agent's answers to batch sub-queries |
| `_subtask_N_input.json` | Input to sub-agent N |
| `_subtask_N_state.json` | Sub-agent N's persistent namespace |
| `_subtask_N_result.json` | Sub-agent N's final answer |
| `_final_answer.json` | Final answer (for long/structured answers) |

All files use `_` prefix to distinguish them from user data files.
