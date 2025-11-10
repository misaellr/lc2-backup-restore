---
name: OpenAI MCP assist
version: 1.0.0
description: >
  When Claude Code encounters unexpected problems, is refining requirements during execution,
  or wants a second opinion, it should consult the configured OpenAI MCP tool, summarize the
  local context, and incorporate the feedback conservatively (KISS; minimal diffs).
owners:
  - platform: github
    handle: misaellr
license: internal
---

# purpose

provide a dependable “second pair of eyes” via the OpenAI MCP tool to (a) unblock execution errors,
(b) crystallize requirements and acceptance criteria while working, and (c) sanity-check designs or code.

# when to invoke this skill

Claude should load and use this skill when any of the following are true:

1. unexpected problem/issue: a tool error, failing test, runtime exception, or blocked dependency.
2. requirements work: defining or refining scope, acceptance criteria, edge cases, or test scenarios while executing a task.
3. doubt: uncertainty between options where a brief critique/alternatives review would reduce risk.

# how to engage the OpenAI MCP tool

if an MCP server named `openai` (or equivalent) is available:

- summarize context succinctly (task, repo path(s), relevant files, prior attempts, error output).
- redact secrets (tokens, passwords, keys) and minimize payloads (only the smallest necessary snippets).
- call the tool with the appropriate intent template below.
- adopt changes conservatively: prefer the smallest, reversible edits; propose unified diffs.
- record rationale in the response so decisions are auditable.

if the OpenAI MCP tool is not available, proceed without it and note “OpenAI MCP unavailable” in the reasoning.

## intent templates

### A) unblock an error
prompt to OpenAI MCP:
- goal: diagnose and propose the minimal change to fix this failure.
- include: error excerpt, smallest reproducible snippet, env sketch (OS, runtime, package versions).
- ask for: brief diagnosis (1–3 bullets), minimal fix (unified diff or code block), and a one-line risk note.

### B) define/refine requirements
prompt to OpenAI MCP:
- goal: refine scope and testability.
- include: current requirement draft, constraints, stakeholders.
- ask for: acceptance criteria (must/should), edge cases, test scenarios (unit/integration), and any dependency risks.

### C) second opinion
prompt to OpenAI MCP:
- goal: critique and alternatives.
- include: current plan/patch/architecture in brief.
- ask for: top risks, 1–2 viable alternatives (trade-offs), and a quick “proceed or revise” recommendation.

# output and formatting rules

- prefer unified diffs for code changes; otherwise, show smallest self-contained snippets.
- provide commit message suggestions:
  - imperative mood, ≤ 72-char subject, brief body with “why / what / impact”.
- attribution policy: do not add OpenAI, Claude, or any AI as a documentation author, copyright
  holder, commit author, or co-author. no “signed-off-by” lines from AI. human authorship only.
- style: KISS; simple things simple, complex things possible. avoid large refactors unless explicitly requested.

# privacy and safety

- do not transmit secrets or large proprietary dumps. mask tokens and credentials.
- respect repository policy and license headers.
- keep vendor-neutral language; no benchmarking claims.

# quick self-test (for Claude)

- can I reproduce the issue from the provided snippet?
- did I redact secrets?
- did I propose the smallest viable change and explain the trade-off?
- did I generate clear acceptance criteria and tests when asked?
- did I avoid adding AI attribution to commits or docs?
