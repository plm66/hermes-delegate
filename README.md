# delegate_task — Named Profile Routing for Hermes Subagents

> **Branch:** `feat/delegate-task-profile`  
> **Base repo:** [NousResearch/hermes-agent](https://github.com/NousResearch/hermes-agent)  
> **Issue:** [#71448](https://github.com/NousResearch/hermes-agent/issues/71448)

---

## What this PR does

This branch extends `delegate_task` with an optional `profile` parameter.  
When provided, the child subagent runs under a **named Hermes profile** — a separate configuration directory with its own provider, model, API credentials, and `SOUL.md` identity — instead of inheriting everything from the parent agent.

Everything else (batch mode, toolset bounding, background dispatch, ACP transport, orchestrator depth) works exactly as before when `profile` is omitted.

---

## Why it matters

Without this change, all subagents share the parent's provider and credentials.  
A task that benefits from a cheaper fast model, a different API key, or a specific persona had no clean way to express that — config hacks or environment mutations were required.

With `profile=`, the agent can delegate to a specialist subagent in one line:

```python
# Single task — route to the "analyst" profile
delegate_task(
    goal="Summarise the quarterly revenue data",
    profile="analyst",
)

# Batch — each task uses a different profile
delegate_task(tasks=[
    {"goal": "Draft the investor memo",    "profile": "writer"},
    {"goal": "Run the financial model",    "profile": "analyst"},
    {"goal": "Translate to Japanese",      "profile": "translator"},
])
```

---

## How profiles work

A profile is a directory under `~/.hermes/profiles/<name>/`:

```
~/.hermes/profiles/analyst/
    config.yaml      ← provider, model, base_url, max_tokens, …
    .env             ← HERMES_INFERENCE_API_KEY, etc.
    SOUL.md          ← identity injected as system-prompt slot #1
```

Create one with:

```bash
hermes profile create analyst
hermes model                    # pick the model for this profile
```

---

## What the `profile` parameter controls

| Setting | Source |
|---------|--------|
| Provider / model / base URL / API key | `config.yaml` via `resolve_runtime_provider()` |
| Secrets (`HERMES_INFERENCE_API_KEY`, …) | `.env` via `build_profile_secret_scope()` |
| Agent identity | `SOUL.md` loaded with `load_soul_identity=True` |

---

## Isolation guarantees

| Property | Guarantee |
|----------|-----------|
| `HERMES_HOME` override | Context-local (ContextVar), always restored in `finally` — both at build time and in the worker thread |
| Secret scope | Same — `set_secret_scope` / `reset_secret_scope` inside englobing `try-finally` |
| ACP transport | Neutralised unconditionally when `profile=` is set, regardless of parent ACP or explicit `delegation.command` override |
| Toolset bounding | Unchanged — profile cannot grant the child more tools than the parent |
| Partial-setup failure | Fail-closed — if scope installation fails mid-setup, the home token is always released and the child never runs with the parent's context |
| Backwards compatibility | When `profile` is omitted, behaviour is strictly unchanged |

---

## Validation

```
name                            format    exists on disk
─────────────────────────────── ──────── ───────────────
validate_profile_name()         ✓        ✓
reserved names (e.g. "test")    rejected
path traversal (../../etc)      rejected
non-existent profile            rejected before any build
```

---

## Files changed

| File | Nature | +/- |
|------|--------|-----|
| `tools/delegate_tool.py` | Implementation | +426 / −95 |
| `tests/tools/test_delegate.py` | Tests | +919 / 0 |
| `run_agent.py` | Dispatch propagation | +1 / 0 |
| `README.md` (this file) | Documentation | new |

---

## Test coverage

```
TestProfileParam   (9 tests)  — schema, dispatch, parameter propagation
TestProfileContract (21 tests) — validation, result metadata, ACP neutralisation,
                                  SOUL.md production path, context-local isolation,
                                  scope-restoration on partial failure (build + worker),
                                  fail-closed guarantees
```

Full run:

```bash
uv run pytest tests/tools/test_delegate.py   # 196 passed
uv run ruff check tools/delegate_tool.py tests/tools/test_delegate.py run_agent.py
git diff --check
```

All three commands exit cleanly.

---

## Commit history

```
64b66d90c docs(delegate): document named-profile subagent routing in README
2d5edef04 test(delegate): add contract and scope-restoration tests for named profiles
6571afbc1 feat(delegate): implement context-local profile routing and validation
97fc1ce0e feat(delegate): propagate profile param to delegate_task
```

---

## Original Hermes README

The upstream Hermes Agent README is preserved at [`README.hermes.md`](README.hermes.md).
