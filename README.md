# rlm-skill

A Claude Code / GitHub Copilot agent skill that implements the **Recursive Language Models (RLM)** reasoning pattern.

Instead of answering in one shot, the agent runs an iterative REPL-style loop — write code → execute in terminal → read results → reason → repeat — approximating `rlm.completion()` from the RLM library.


## What's Inside

```
SKILL.md                     ← Core skill: triggers, primitives table, 5-step workflow, rules
references/
  primitives.md              ← Full code for every primitive (llm_query, rlm_query, FINAL_VAR, …)
  reasoning-pattern.md       ← Chunking strategies, error tracking, computation chaining, cleanup
```

## Installation

Copy this directory into your workspace under `.claude/skills/rlm-pattern/`:

```bash
cp -r rlm-skill/ /your/project/.claude/skills/rlm-pattern/
```

Claude Code/Vscode github copilot automatically discovers skills under `.claude/skills/` — no registration needed. Restart your session after adding the skill.

The skill activates when the agent detects trigger phrases such as:
- "reason iteratively over this"
- "use RLM pattern"
- "analyze this large context"
- "decompose this task with code"
- "iterate until you find the answer"

See [`SKILL.md`](SKILL.md) for the full trigger list and workflow.

## Example

```
User: Use RLM pattern to analyze all Python files in this repo and find duplicated logic

Agent: TASK: Find duplicated logic across all Python files
       MAX_ITERATIONS: 10
       ITERATIONS_USED: 0
       [iter 1] listing files and measuring sizes...
       [iter 2] spawning 4 subagents to analyze each module in parallel...
       [iter 3] aggregating results...
       FINAL_ANSWER: Duplicated logic found in X and Y — both implement the same ...
```

## What the Agent Does

1. Declares a session plan (`TASK / MAX_ITERATIONS / ITERATIONS_USED`)
2. Loads context into `_rlm_state.json` on iteration 0
3. Runs Python scripts in terminal each iteration — deterministic computation in code, language reasoning from stdout
4. Uses `llm_query_batched` for parallel semantic sub-queries (stdout reasoning when possible, file handoff as fallback)
5. Uses `rlm_query` (native subagent spawn) for sub-tasks needing their own iteration loop
6. Signals `FINAL_ANSWER:` exactly once, then cleans up working files

## Limitations

This skill approximates the RLM library pattern in agent mode. Compared to the full `rlms` package:

- **No hard budget/token enforcement** — iteration limits are best-effort (the agent self-regulates, not infrastructure)
- **No remote sandbox support** — code runs in the local terminal, not Docker/Modal/E2B
- **No persistent REPL namespace** — state is persisted via `_rlm_state.json` between terminal runs instead

For production workloads requiring strict resource limits or remote execution, use the [rlm library](https://github.com/alexzhang13/rlm) directly.

## Upstream Source

This skill adapts the [rlm](https://github.com/alexzhang13/rlm) library (package: `rlms`, [arXiv:2512.24601](https://arxiv.org/abs/2512.24601)) for agent-native use. The original system prompt lives in `rlm/utils/prompts.py → RLM_SYSTEM_PROMPT`.

## License

MIT
