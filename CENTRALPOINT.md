# CENTRALPOINT — Hermes `delegate_task(profile=...)`

## 0.5 Doctrine

Ce fichier est le journal de coordination de ce worktree. Toute action
substantielle doit être précédée de la lecture des travaux en cours et suivie
d'une entrée horodatée dans le journal. Les secrets et leurs valeurs ne sont
jamais consignés.

## 3. Work in Progress

- Aucun travail d'écriture actif. La validation E2E du routage par profil est
  terminée sur `feat/delegate-task-profile`.

## 5. Journal de Dialogue

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
