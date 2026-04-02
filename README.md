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

## Usage

Place this skill in your VS Code / Claude Code workspace under `.claude/skills/rlm-pattern/` (or any path your agent skill loader is configured to read from).

The skill activates when the agent detects trigger phrases such as:
- "reason iteratively over this"
- "use RLM pattern"
- "analyze this large context"
- "decompose this task with code"
- "iterate until you find the answer"

See [`SKILL.md`](SKILL.md) for the full trigger list and workflow.

## What the Agent Does

1. Declares a session plan (`TASK / MAX_ITERATIONS / ITERATIONS_USED`)
2. Loads context into `_rlm_state.json` on iteration 0
3. Runs Python scripts in terminal each iteration — deterministic computation in code, language reasoning from stdout
4. Uses `llm_query_batched` (file handoff) for parallel semantic sub-queries
5. Uses `rlm_query` (native subagent spawn) for sub-tasks needing their own iteration loop
6. Signals `FINAL_ANSWER:` exactly once, then cleans up working files

## Upstream Source

This skill adapts the [rlm](https://github.com/alexzhang13/rlm) library (package: `rlms`, [arXiv:2512.24601](https://arxiv.org/abs/2512.24601)) for agent-native use. The original system prompt lives in `rlm/utils/prompts.py → RLM_SYSTEM_PROMPT`.

## License

MIT
