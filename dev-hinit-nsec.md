---
required-reading:
  - "[[dev-knw-nsec-rules]]"
---

# NUU Secrets (nsec) — Headless

This is the init of the NUU Secrets shard for a headless Orbh session. The rules are the same as in an interactive session. The difference: no person answers you during the turn. A missing value, a missing key, or a leak is a blocker. You report a blocker. You never guess.

## What NUU Secrets Is

NUU Secrets keeps the secrets of a team in one encrypted Git store (SOPS + age) and gives them to programs. The CLI is `secrets`. Its alias `secret <cmd>` runs a program with the values in its process environment. It writes no file with a plaintext value (a refresh or a first clone changes the Git files and the index of the store clone). Each project has a committed `secrets.toml` that maps variables to secret names (`secret://clerk/secret-key`) and literals. The environments are `dev`, `preview` (dev with preview values over it), `prod` (never falls back to dev), and `deploy`. Only the operator enters values and manages keys. Before a store commit, the CLI scans the exact bytes of the change unless gitleaks is missing or `SECRETS_GITLEAKS=0`. The model is in [[dev-knw-nsec-model]]. The commands and the known limits are in [[dev-knw-nsec-cli]].

## The Rules in Short

The complete rules are in [[dev-knw-nsec-rules]].

1. Never print, log, echo, or type a value. This includes your result, an Orbh message, an update, the `blockers` key, a Mesh note, a commit message, and a test fixture.
2. Never write a value to a file. Never pass a value as a process argument.
3. Never read or copy the key file. Never decrypt or edit a store file. `secrets keygen --public` (with no `--file`) gives you the public key of this machine.
4. Never run an operator-only command (`secrets set`, `edit`, `unset`, `keygen` except `keygen --public` with no `--file`, `recipient add|rm`, `store init`, `store fix-config`, `store remove`, `push`, `login`, `logout`). The one exception: your prompt relays an instruction of the operator that names that exact command. Never add `--yes` to a push.
5. Never set `SECRETS_GITLEAKS=0`, and never remove, rename, or hide gitleaks on `PATH`. Before a store write that this shard allows, check for an inherited `SECRETS_GITLEAKS=0`. See [[dev-knw-nsec-rules]] § Rule 10.
6. You are a user. The rules prevent accidents. They are not a security boundary.

## What Is Different in a Headless Session

1. **No interactive prompt.** Never start a command that waits for a person: `secrets set` (a hidden prompt), `secrets edit` (an editor), or `secrets login` (a browser). These are operator commands.
2. **A blocker goes to a person, never to a guess.** These are blockers:
   - a value is missing (`✘ <n> variable(s) have no value in env <env> …`, or a `doctor` line `… is not in the store`);
   - `secret` stops with `✘ No usable age key: …`, or `doctor` says `no age key file at <path>` or `this machine is not a recipient of the store`;
   - `--offline` stops with `… has no cached clone on this machine` and the task forbids network access;
   - a `doctor` item that only the operator can fix ([[dev-sk-nsec-diagnose]]);
   - a tool needs a file with values;
   - a push to a platform: only the operator runs `secrets push` ([[dev-sk-nsec-plan_push]]);
   - a store write stops with the secret scan (`SECRET_SCAN_FOUND` or `SECRET_SCAN_FAILED`) or with the store policy (`UNSAFE_STORE` or `UNSAFE_RESULT`: a field name, never a value), or its message has a cleanup line or `Nothing was pushed.` ([[dev-sk-nsec-diagnose]], step 9);
   - the environment of a store write that you may run has `SECRETS_GITLEAKS=0` (the `doctor` line `SECRETS_GITLEAKS=0: the CLI does not scan store commits`).

   For a blocker, do not invent a value. Do not use a placeholder that looks real. Do not remove the variable from the mapping to make the run pass. Do not skip the check and report success. Do not turn off the secret scan. Write the request with [[dev-sk-nsec-request_value]] (a value or a key) or with the report of [[dev-sk-nsec-diagnose]] (an operator fix, also a scan refusal), set the `blockers` key, and deliver it as in [Result Delivery](#result-delivery).

   A store write that printed `secrets: gitleaks is not on PATH, …` had no scan. Check its exit status, and report both facts in your result.
3. **Work that does not need the value goes on.** Finish the parts of the task that do not need the missing value. Then return with the blocker.
4. **Background programs.** Run a dev server under `secret` in the background of your harness. Stop it before you return. The process of the turn ends with the turn.
5. **A leak is a blocker of the highest priority.** Stop. Do not repeat the value. Report the variable, the secret name, the environment, and the place of the leak. Say that the session transcript holds the value. Follow [[dev-knw-nsec-rules]] § If a value leaks.

## Orbh Interface Schema

Use the session keys so that the dispatcher and the operator see the state without the transcript. Never put a value in a key. Names, environments, and paths are public.

| Key | Value | When |
|---|---|---|
| `phase` | One of the phase values below | When NUU Secrets work is the main work of the turn. Else keep the phase of your main task. |
| `blockers` | The secret names and environments that need the operator, for example `clerk/secret-key (dev), sentry/dsn (dev)` | When a blocker starts. Set it to `none` when the blocker is gone. |

```sh
flint orbh session set phase secrets-blocked
flint orbh session set blockers "clerk/secret-key (dev): no value in the store"
```

### Phase Values

| Phase | Meaning |
|---|---|
| `secrets-setup` | You write `secrets.toml` and `.gitignore` ([[dev-sk-nsec-setup_project]]). |
| `secrets-run` | You run a program with `secret` ([[dev-sk-nsec-run_app]]). |
| `secrets-diagnose` | You run `secrets doctor` and act on it ([[dev-sk-nsec-diagnose]]). |
| `secrets-plan` | You run `secrets plan` and prepare the commands for the operator ([[dev-sk-nsec-plan_push]]). |
| `secrets-blocked` | You wait for the operator: a value, a key, a push, or an operator fix. |

## Result Delivery

- **A subagent** puts each blocker in its result for the dispatcher. Only a root or a manager session contacts the operator. End the turn with:
  ```sh
  flint orbh session return --finish "<result: what is done, and the request of request_value>"
  ```
- **A headless root session** sends the request to the operator. Then it ends the turn with `--await`, because it expects the answer of the operator:
  ```sh
  flint orbh approval request "<title>" --body "<the request of request_value>"
  flint orbh session return --await "<result: what is done, and the blocker>"
  ```
  The approval request does not end the turn. Only `return` ends it.
- The result has names, environments, commands, and `doctor` lines. It never has a value.
- A leak goes first in the result, with the variable, the secret name, the environment, and the place.

## Situations

| Situation | Use |
|---|---|
| Run an app, a test, or a script with its secrets | [[dev-sk-nsec-run_app]] |
| A project has no `secrets.toml`, or a variable has no entry | [[dev-sk-nsec-setup_project]] |
| `secret` or `secrets` fails, or access is not correct | [[dev-sk-nsec-diagnose]] |
| A value or a key is missing | [[dev-sk-nsec-request_value]], then return with the blocker |
| Prepare a push to Vercel, Convex, or Railway | [[dev-sk-nsec-plan_push]], then return with the commands for the operator |
