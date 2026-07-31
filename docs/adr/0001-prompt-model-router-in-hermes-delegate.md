# ADR-0001 — Prompt→Model router as a module of the `hermes-delegate` package

Status: **Proposed** (recorded before implementation)
Date: 2026-08-03
Author: delegation worktree (`feat/delegate-task-profile`)
Scope: future standalone package `hermes-delegate`; zero footprint on hermes-agent core
Supersedes: none (the reserved-but-unimplemented `smart_model_routing` config key in hermes-agent is left untouched)

## Context

The operator wants a tool that reads each prompt and selects the model best suited to
answer it. Grounding facts verified in the hermes-agent checkout:

1. **`smart_model_routing` is a dead reserved key.** `hermes_cli/config.py:5739`
   whitelists the root key; the setup wizard writes `enabled: False`
   (`hermes_cli/setup.py:3094`). **No engine, no reader, no resolver exists anywhere
   in the tree** — the key is a placeholder waiting for a decision on *where* the
   feature lives.
2. **Core constraints (AGENTS.md).**
   - *Per-conversation prompt caching is sacred.* Model swaps inside a live
     conversation invalidate the cached prefix (cache is keyed by model + prefix);
     mid-turn swaps additionally break tool-call continuation. The only sanctioned
     in-loop context mutation is compression.
   - *Narrow waist.* New model tools ship on every API call; capability belongs at
     the edges (CLI + skill → service-gated tool → plugin → MCP → core tool, last).
   - *Standalone plugin repos* are the sanctioned home for niche capability
     (memory providers, third-party products). In-tree core integration is the
     exception, not the default.
   - Plugins MUST NOT modify core files; if a plugin needs more surface, the
     *generic* plugin surface is widened — never plugin-specific core code.
3. **This worktree's mission is delegation.** `feat/delegate-task-profile` ships
   `delegate_task(profile=...)` named-profile routing (`tools/delegate_tool.py`),
   and `CENTRALPOINT.md` coordinates it. The natural next step is a standalone
   `hermes-delegate` package that owns the delegation surface: named profiles,
   subagent lifecycle, task dispatch, per-profile provider/model scoping.
4. **Reusable infrastructure exists in core.** `agent/auxiliary_client.py::_resolve_auto`
   already implements a provider→model resolution chain (main provider → OpenRouter →
   portal → custom endpoint → native Anthropic → direct keys) and the `auxiliary:`
   config section pins per-task models. The router must **reuse** this machinery,
   not duplicate it.

## Decision

Build the prompt→model router as **`hermes_delegate.model_router` — a module inside
the future `hermes-delegate` package**, operating at **delegation boundaries**, not
inside the hermes-agent conversation loop.

### Core insight that drives the decision

**Routing at delegation/spawn time is cache-safe by construction.** Each subagent is
a fresh conversation context: choosing the (provider, model, profile) for a task at
the moment the subagent is spawned costs **zero** cache invalidation and cannot break
message-role alternation. The main-loop router would fight the prompt-cache invariant
on every switch; the delegator-side router never touches it. Model selection belongs
where the context is born, not where it lives.

### Surface 1 — Spawn-time routing (primary, shipped first)

`hermes-delegate` intercepts delegation requests and classifies each task prompt
*before* spawning the worker:

```
task prompt → classifier → (profile, provider, model) → delegate_task(profile=…)
```

- Classification is **heuristics-first and deterministic**: prompt length, attached
  images/modality, language, task keywords (code/refactor → code tier, summarization →
  small tier), required toolsets, and the target profile's own model config.
- Optional **cheap-LLM classifier** (fast/small model, e.g. the auxiliary chain) is
  invoked **only when heuristics are ambiguous** — never as the default path. Its
  verdict is cached per prompt-shape (hash of normalized prompt + toolset signature).
- Config lives in hermes-delegate's own config (tiers, rules, cost ceiling,
  `max_switch_per_session`), scoped per profile. hermes-agent's `DEFAULT_CONFIG` and
  the `smart_model_routing` key are **not** touched.
- Resolution reuses the `_resolve_auto` chain semantics so a tier can resolve to
  "auto" and inherit the user's provider fallback order.

### Surface 2 — Main-loop bridge (parked, deferred)

If the operator later wants routing in the *main* session too, the bridge is a
widened **generic plugin hook** (`on_turn_start` → optional model override, applied
only at turn boundaries, never mid-turn) requested as a core change *separately and
on its own merits* — not as part of hermes-delegate. Until that hook exists, the
module ships spawn-time routing only. No hermes-agent source file is modified by
hermes-delegate.

## Consequences

### Positive

- **Zero core footprint.** No core tool, no new env var, no `DEFAULT_CONFIG` edit.
  The reserved `smart_model_routing` key stays untouched; if the main-loop engine is
  ever built, this module remains a valid edge consumer of it.
- **Cache-safe by construction.** No mid-conversation model swap, ever.
- **Profile-native.** Each profile keeps its own provider/model; the router selects
  among them instead of imposing one. Matches the delegation story
  (`delegate_task(profile=…)`) end to end: choose the *right worker* and the *right
  model* for the *right task*.
- **Standalone evolution.** The package can move, version, and test independently;
  aligns with the standalone-plugin-repo policy.

### Negative / risks

- **Not available to non-delegation workflows** out of the box; main-session routing
  depends on the parked Surface-2 hook.
- **Two sources of truth for "which model"** (profile config + router overrides) must
  be reconciled explicitly: profile-pinned models are authoritative unless a router
  rule overrides them, and the override must be visible in the delegation audit trail.
- **Classifier cost/quality.** The LLM classifier must stay cheap and cached;
  heuristics must cover the common cases so the LLM path is rare.
- **Scope creep.** The module is a *model selector*, not a task planner. Any ambition
  beyond (provider, model, profile) selection is rejected in this ADR.

## Alternatives considered

1. **Core engine for `smart_model_routing` in hermes-agent.** Rejected. Footprint on
   the narrow waist, and the main-loop swap invalidates the prompt cache per switch —
   the design fights the project's #1 invariant. Also niche: routing matters most in
   delegation fleets, not single-session use.
2. **General plugin in hermes-agent (existing hooks).** Rejected as the primary
   home. Existing hooks are observers (`pre_llm_call` et al.) and cannot swap
   `self.model` at turn boundaries; making them authoritative requires widening the
   plugin surface in core anyway (a core change), and it does not serve the delegate
   use case, which spawns fresh agents rather than mutating a live one.
3. **Skill-only (agent self-selects its model).** Rejected. Non-deterministic, no
   guarantee, and model selection is exactly the kind of decision that must not be
   left to the model being selected.
4. **CLI wrapper respawning `hermes` with different flags per task.** Rejected.
   Heavy, loses session state, and does not compose with `delegate_task`.
5. **MCP server in the catalog.** Rejected. MCP is for tool-shaped I/O the agent
   invokes; routing is infrastructure the delegator owns, not a tool call.

## References

- `hermes_cli/config.py:5739` — reserved root key `smart_model_routing`
- `hermes_cli/setup.py:3094` — wizard writes `enabled: False`
- `tools/delegate_tool.py` — `delegate_task(profile=…)` (this worktree)
- `agent/auxiliary_client.py::_resolve_auto` — resolution chain to reuse
- `agent/turn_context.py` — per-turn model accounting (relevant for Surface 2)
- `AGENTS.md` — prompt-caching invariant, footprint ladder, standalone-plugin policy
- `CENTRALPOINT.md` — worktree coordination journal
