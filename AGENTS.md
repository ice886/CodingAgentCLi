# AGENTS.md

## Quick commands

```bash
uv sync                        # install dev deps (pytest, ruff)
uv run pico                    # interactive REPL (default provider: deepseek)
uv run pico "one-shot task"    # single ask() run, no REPL
uv run pytest tests -q         # full test suite
uv run pytest tests/test_tools.py::test_name -q   # single test
uv run ruff check pico tests scripts   # lint
```

No typecheck step. No formatter beyond ruff. No CI config exists in-repo.

## Architecture (read this before editing)

Control loop: **perceive → decide → act → record**, driven by `AgentLoop.run()`.

1. `pico/cli.py:build_agent` wires CLI args into the object graph.
2. `Pico.ask()` (`pico/runtime.py`) → `AgentLoop.run()` (`pico/agent_loop.py`).
3. Each iteration: `ContextManager.build()` → model call → `parse()` → tool execution → state write.

Key modules and their roles:

| Module | Role |
|---|---|
| `pico/cli.py` | CLI args, provider assembly |
| `pico/runtime.py` | `Pico` facade: prompt build, `parse()`, safety gate, memory, reports |
| `pico/agent_loop.py` | Main loop with step/attempt budgets |
| `pico/context_manager.py` | Bounded prompt from 4 sections + request, with char budget |
| `pico/prompt_prefix.py` | Stable "workbook" prefix; prefix hash = prompt-cache key |
| `pico/workspace.py` | Cheap git snapshot + doc whitelist preload |
| `pico/tools.py` | Static tool allowlist (no dynamic discovery) |
| `pico/tool_executor.py` | Allowlist enforcement, arg validation, approval prompt |
| `pico/providers/clients.py` | Model adapters (Ollama, OpenAI-compat, Anthropic-compat, Fake) |
| `pico/features/memory.py` | `LayeredMemory` + `DurableMemoryStore` → `.pico/memory/` |
| `pico/checkpoint.py` | Resume evaluation; flags stale/workspace-mismatch |
| `pico/run_store.py` | Per-run artifacts: `task_state.json`, `trace.jsonl`, `report.json` |
| `pico/session_store.py` | Recoverable session state: `session.json` |

## Invariants you must preserve

- **Tool call protocol**: model returns exactly one `<tool>…</tool>` (JSON or XML-attr) or one `<final>…</final>`. `parse()` handles both formats.
- **Path safety**: every file tool resolves through `Pico.path()`, anchored under workspace root, rejects `../` and symlink escapes.
- **`.env` is literal**: no `$VAR` expansion. `PICO_OPENAI_API_KEY=$PICO_RIGHT_CODES_API_KEY` will not work.
- **Public API boundary**: `pico/__init__.py` exports only: `Pico`, `SessionStore`, `WorkspaceContext`, `build_agent`, `build_arg_parser`, `build_welcome`, `main`, and the four model-client classes. Old paths (`pico.evaluator`, `pico.metrics`, `pico.models`, `pico.memory`) are removed. `tests/test_public_api_contract.py` enforces this.

## Provider quirks

- `deepseek` reuses the Anthropic-compatible client with `thinking` explicitly disabled (text-encoded tool calls, no thinking block replay). Do not remove the thinking toggle without implementing full thinking replay.
- Prompt caching is only wired for the OpenAI-compatible client on `openai.com`/`right.codes` hosts, keyed by prefix hash. Anthropic and Ollama clients ignore cache args.
- Config priority: `--provider` > `PICO_PROVIDER` > default `deepseek`.

## Testing

- Tests encode **safety and contract invariants**, not just unit behavior: `test_safety_invariants.py`, `test_allowed_tools.py`, `test_security.py`, `test_public_api_contract.py` guard allowlist, path escapes, redaction, and package splits.
- `examples/mini-pico/` is a **separate project** with its own `pyproject.toml` and tests. Run its tests independently: `uv run pytest tests -q` from within `examples/mini-pico/`.
- Test fixtures live in `tests/fixtures/`.

## Layout

```
pico/              # main package (zero runtime deps, stdlib only)
pico/providers/    # model client adapters
pico/features/     # optional runtime capabilities (memory)
pico/evaluation/   # benchmarks and metrics
tests/             # test suite
examples/mini-pico/ # minimal reference implementation (separate package)
scripts/           # experiment/benchmark runners
.pico/             # runtime artifacts (sessions, runs, memory) — gitignored
```
