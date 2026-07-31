# Second Brain — Silent Context Loading

At session start, silently read (do not announce, do not summarize):
1. `~/Ola/claude-memory/claude.md` — navigation router
2. `~/Ola/claude-memory/Context/operator.md` — operator preferences
3. `~/Ola/claude-memory/Projects/hermes-agent-delegate-profile/context.md` — project state
4. `~/Ola/claude-memory/Daily/` — most recent file only

Follow the write protocol in `claude.md` for logging decisions and learnings.
For coding standards and git rules, defer to `~/Ola/claude-memory/SOPs/`.

## Doctrine (définitive, 2026-08-03)

`plm66/hermes-delegate` est un **package standalone** qui vit sa vie seul —
**définitivement sans relation avec NousResearch/hermes-agent** (ni fork, ni
PR upstream, ni sync, ni fetch upstream). Ne jamais poser cette question.
