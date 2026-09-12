# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

`pico` is a lightweight local coding agent harness: it wraps a model in a control loop, gives it a constrained tool allowlist for reading/editing a repository, and persists session + per-run audit artifacts under `.pico/`. Python 3.10+, **zero runtime dependencies** (stdlib only: `urllib`, `subprocess`, `json`).

## Commands

```bash
uv sync                        # install dev deps (pytest, ruff)
uv run pico                    # interactive REPL (default provider: deepseek)
uv run pico "one-shot task"    # single ask() run, no REPL
uv run pytest tests -q         # full test suite
uv run pytest tests/test_tools.py::test_name -q   # single test
uv run ruff check pico tests scripts   # lint
```

Config comes from `.env` (see `.env.example`); it must be copied manually and real keys are never committed.

## Architecture

The runtime is a **perceive → decide → act → record** loop around a model. The flow for one user request:

1. `pico/cli.py:build_agent` wires CLI args into an object graph: `WorkspaceContext`, a model client, a `SessionStore`, and a `Pico` instance.
2. `Pico.ask()` (in `pico/runtime.py`) delegates to `AgentLoop.run()` (`pico/agent_loop.py`), the actual control loop.
3. Each iteration builds a prompt via `ContextManager.build()` (`pico/context_manager.py`), requests the model, parses the response, executes any tool call, and writes state.

Key modules (big-picture roles):

- **`pico/cli.py`** — argument parsing and assembly. Provider selection priority: `--provider` > `PICO_PROVIDER` > default `deepseek`.
- **`pico/runtime.py`** — `Pico` facade. Owns prompt-building, `parse()` (raw model text → `tool`/`final`/`retry`), the tool-execution safety gate, memory updates, and report/trace writing. Many concerns are delegated to the modules below.
- **`pico/agent_loop.py`** — `AgentLoop.run()`, the main loop with step/attempt budgets, stop conditions, and artifact persistence.
- **`pico/context_manager.py`** — assembles the bounded prompt from four sections (`prefix`, `memory`, `relevant_memory`, `history`) plus the current request, with a total char budget and a fixed reduction order.
- **`pico/prompt_prefix.py`** — builds the stable "workbook" prefix (identity, rules, tool signatures, examples, workspace snapshot) and hashes it. The prefix hash doubles as the prompt-cache key.
- **`pico/workspace.py`** — `WorkspaceContext.build()` collects a cheap git snapshot (branch, status, recent commits) and preloads only a whitelist of docs (`AGENTS.md`, `README.md`, `pyproject.toml`, `package.json`), not the whole repo.
- **`pico/tools.py`** — the explicit tool allowlist (`list_files`, `read_file`, `search`, `run_shell`, `write_file`, `patch_file`, plus bounded `delegate`). Tools are registered statically, not discovered dynamically.
- **`pico/tool_executor.py`** — `ToolExecutor.execute()` is the enforcement layer: allowlist check, arg validation, repeated-call rejection, approval prompt for risky tools, and workspace diff snapshots.
- **`pico/providers/clients.py`** — model adapters (`OllamaModelClient`, `OpenAICompatibleModelClient`, `AnthropicCompatibleModelClient`, `FakeModelClient`) that flatten HTTP/SSE/usage differences into a uniform `complete()` returning text and `last_completion_metadata`.
- **`pico/features/memory.py`** — `LayeredMemory` (working memory: task summary, recent files, file summaries, episodic notes) plus `DurableMemoryStore`, which persists durable notes to `.pico/memory/MEMORY.md` + `topics/`.
- **`pico/checkpoint.py`** — resume-state evaluation and checkpoint creation; compares saved runtime identity against the current one to flag stale/workspace-mismatch resumes.
- **`pico/run_store.py` / `pico/session_store.py`** — per-run artifacts (`task_state.json`, `trace.jsonl`, `report.json`) vs. recoverable session state (`session.json`).
- **`pico/task_state.py`** — `TaskState` state machine (`running → completed/stopped/failed`) with stop reasons.
- **`pico/config.py`, `pico/security.py`** — `.env` parsing and secret detection/redaction.

## Public API boundary

`pico/__init__.py` exports only the current names: `Pico`, `SessionStore`, `WorkspaceContext`, `build_agent`, `build_arg_parser`, `build_welcome`, `main`, and the four model-client classes. The old `pico.evaluator`, `pico.metrics`, `pico.models`, and `pico.memory` import paths are **removed** — use `pico/evaluation/`, `pico/providers/`, and `pico/features/` instead. `tests/test_public_api_contract.py` enforces this.

## Important invariants

- **Tool call protocol**: the model must return exactly one `<tool>…</tool>` (JSON or XML-attr form) or one `<final>…</final>`. `parse()` in `runtime.py` handles both tool formats; multi-line `write_file`/`patch_file` prefer the XML form.
- **Path safety**: every file tool resolves paths through `Pico.path()`, which anchors them under the workspace root and rejects `../` escapes and symlink escapes.
- **Provider details**: `deepseek` reuses the Anthropic-compatible client with `thinking` explicitly disabled (Pico uses text-encoded tool calls and does not replay native thinking blocks). Prompt caching is only wired for the OpenAI-compatible client on `openai.com`/`right.codes` hosts, keyed by the prefix hash; the Anthropic and Ollama clients ignore cache args.
- **`.env` is literal**: the parser does not expand `$VAR` references, so `PICO_OPENAI_API_KEY=$PICO_RIGHT_CODES_API_KEY` will not work — set each variable to a literal value.
- **`delegate` tool** spawns a read-only child `Pico` with `approval_policy="never"` and reduced `max_steps`, depth-bounded (`max_depth`), returning only its conclusion text.

## Tests

The suite encodes safety and contract invariants, not just unit behavior: `test_safety_invariants.py`, `test_allowed_tools.py`, `test_security.py`, and `test_public_api_contract.py` guard the allowlist, path-escape protection, redaction, and the package split. `examples/mini-pico/` is a separate, minimal reference implementation with its own tests.
