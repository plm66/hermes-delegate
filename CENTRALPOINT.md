# CENTRALPOINT — Hermes `delegate_task(profile=...)`

## 0.5 Doctrine

Ce fichier est le journal de coordination de ce worktree. Toute action
substantielle doit être précédée de la lecture des travaux en cours et suivie
d'une entrée horodatée dans le journal. Les secrets et leurs valeurs ne sont
jamais consignés.

### Doctrine — relation avec NousResearch (2026-08-03, définitive)

- `plm66/hermes-delegate` est un **package standalone** qui vit sa vie seul.
  Il n'a **définitivement rien à voir** avec NousResearch/hermes-agent : ni
  fork, ni dépendance, ni PR upstream, ni synchronisation.
- Le code hermes-agent présent dans le dépôt est un **seed/snapshot** importé
  comme base de travail — pas une affiliation.
- **Ne plus jamais poser la question** « est-ce qu'on doit PR / sync / aligner
  avec NousResearch ? ». La réponse est NON par définition, pour toujours.
- Conséquence pratique : pas de `git fetch upstream`, pas de PR cross-repo,
  pas de rebase sur upstream/main. Le seul remote de référence est `origin`.
- Décisions liées : ADR-0001 (module du package hermes-delegate), journal 2026-08-03.

## 3. Work in Progress

- ADR-0001 (proposé) : routeur prompt→modèle comme module du futur package
  `hermes-delegate`, opéré aux frontières de délégation (spawn-time), zéro
  footprint core. Fichier : `docs/adr/0001-prompt-model-router-in-hermes-delegate.md`.
  En attente de validation avant implémentation.

## 5. Journal de Dialogue

### 2026-08-03 — Codex — Doctrine : repo standalone, définitivement séparé de NousResearch

- Demande : « notre repo n'a rien à voir — définitivement — avec NousResearch,
  c'est un module package destiné à vivre sa vie seul ; je ne veux plus qu'on
  repose cette question ».
- Vérifié : `plm66/hermes-delegate` est `fork: False` sur l'API GitHub (repo
  indépendant, pas de parent). La PR #1 est interne (feat → main).
- Doctrine consignée dans la section 0.5 ci-dessus : jamais de fetch upstream,
  jamais de PR/sync/alignement avec NousResearch. Question close pour toujours.
- Fichiers touchés : `CENTRALPOINT.md` (0.5 + journal), `CLAUDE.md` (worktree),
  vault Ola (context.md, decisions.md), `Intelligence/recurring-patterns.md`.

### 2026-08-03 — Codex — ADR-0001 : routeur prompt→modèle dans hermes-delegate

- Demande : « avoir la perspective dans l'ADR de construire cet outil en module
  intégré au futur package hermes-delegate ».
- Constat code : `smart_model_routing` est une clé réservée mais morte
  (`config.py:5739`, setup écrit `enabled: False` à `setup.py:3094`) ; aucun
  moteur n'existe. `agent/auxiliary_client.py::_resolve_auto` fournit déjà la
  chaîne de résolution provider→modèle à réutiliser.
- Décision consignée : module `hermes_delegate.model_router`, routage au
  spawn-time des sous-agents (cache-safe par construction — contexte frais),
  heuristiques d'abord + classifieur LLM cheap optionnel et caché ; bridge
  main-loop différé derrière un hook plugin générique élargi (jamais de modif
  core).
- Fichiers : `docs/adr/0001-prompt-model-router-in-hermes-delegate.md` (créé),
  ce `CENTRALPOINT.md` (journal).

### 2026-07-31 16:05 CEST — Codex — Validation E2E avec un profil réel

- Demande : « lancer le test E2E avec un vrai profil pour valider la feature,
  puis consolider le rapport dans CENTRALPOINT.md ».
- Worktree : `/Users/erasmus/DEVELOPER/hermes-agent-delegate-profile` ; branche
  `feat/delegate-task-profile` à `1e234c74c`.
- Prévol : le profil existant `free` a résolu un fournisseur, un modèle, une
  URL et une clé dans son scope local ; aucune valeur secrète n'a été affichée.
- Contrat local : `TestProfileParam` et `TestProfileContract` ont passé
  (`37 passed in 8.90s`).
- E2E : un appel synchrone de `delegate_task(..., profile="free")` a construit
  puis exécuté le sous-agent réel sans outil. Résultat : `completed`, profil
  retourné `free`, 1 appel API, marqueur de réponse attendu présent ; les
  overrides `HERMES_HOME` et le scope de secrets ont été restaurés après
  exécution.
- Contrôles finaux : `ruff check tools/delegate_tool.py tests/tools/test_delegate.py
  run_agent.py` et `git diff --check` ont passé.
- Fichiers touchés : ce `CENTRALPOINT.md` uniquement. Les journaux, session et
  traces live normaux ont été écrits hors dépôt sous `~/.hermes/` par le profil.
- Point non couvert : le test a forcé le chemin synchrone pour observer le
  résultat terminal ; la livraison asynchrone par gateway et la sélection de
  l'outil par un modèle parent ne sont pas testées ici. La propagation du
  dispatcher est couverte par le test de contrat ciblé.
- Avertissement d'environnement : SQLite 3.47.1 a signalé le warning WAL-reset
  connu ; il n'a pas bloqué le scénario et ne provient pas de cette branche.
