---
required-reading:
  - "[[dev-knw-nsec-rules]]"
---

# NUU Secrets (nsec) — Headless

This is the init of the NUU Secrets shard for a headless Orbh session. The rules are the same as in an interactive session. The difference: no person answers you during the turn. A missing value, a missing key, or a leak is a blocker. You report a blocker. You never guess.

## What NUU Secrets Is

NUU Secrets keeps the secrets of a team in one encrypted Git store (SOPS + age) and gives them to programs. The CLI is `secrets`. `secrets pull` writes the values of the target `local` (`dev` or `preview`, never `prod` or `deploy`) into one block of the `.env.local` of a project, and keeps the other lines of the file. Then the program runs with its normal command (`pnpm dev`) and reads `.env.local`. The direction is one way: to change a value, change the store, then pull again. `.env.local` is the one file with plaintext values that the CLI writes (a refresh or a first clone also changes the Git files and the index of the store clone). Since `0.4.0-beta.0`, the CLI has no `run` command and no short alias that starts a program. Each project has a committed `secrets.toml` that maps variables to secret names and literals. A secret name is `<project>/<VARIABLE>`: an entry with no `ref` is an implicit ref to its own project (`nuu-track/CLERK_SECRET_KEY`), and a link names `shared` (`secret://shared/SENTRY_DSN`), the project of the one copy of a value that many projects use. The environments are `dev`, `preview`, `prod`, and `deploy`; each one reads only its own store file (`preview` does not read `dev`). Each secret declares its environments (by default `dev`, `preview`, `prod`; a platform credential `deploy`): a value in an undeclared environment is not delivered. Only the operator enters values and manages keys. One machine can have several identities (names for age key files, in `identities.toml`); for each command, the CLI selects one key file, and you use that selection. `projects.toml` (the project index) lists the projects of this machine and their identities, never values. `secrets ui` is a local web page for the operator; you never use it. Tags (for example `#convex/deploy-keys`, with records in `tags.toml`) label the secrets and the vault entries. A store of the previous version (store format 1: no `store.toml`) needs the migration, and the v2 CLI refuses its reads and its writes (`STORE_FORMAT`); the guide is [[dev-knw-nsec-migrate_v2]]. The operator can give a secret an expiry date in each environment (`metadata.toml`); the CLI warns when it is near (`due`) or past (`expired`). Each store write adds its changes (names, never values) to the change log `changes.jsonl`; when a ref has no value, the CLI follows the log and says `moved to …` or `removed …`. The vault keeps free text (passwords, logins) apart from the environments; you never read or write a vault text. Before a store commit, the CLI scans the exact bytes of the change unless gitleaks is missing or `SECRETS_GITLEAKS=0`. The model is in [[dev-knw-nsec-model]]. The commands and the known limits are in [[dev-knw-nsec-cli]].

## The Rules in Short

The complete rules are in [[dev-knw-nsec-rules]].

1. Never print, log, echo, or type a value. This includes your result, an Orbh message, an update, the `blockers` key, a Mesh note, a commit message, and a test fixture.
2. Never write a value to a file. Only `secrets pull` writes values to a file: the block of `.env.local` (see 12). Never pass a value as a process argument.
3. Never read or copy a key file (also the key file of an identity). A read that stops with `… has SOPS metadata that the CLI does not decrypt. …` is a blocker. Never decrypt or edit a store file, a vault file, `tags.toml`, `metadata.toml`, `changes.jsonl`, `identities.toml`, or `projects.toml`. `secrets keygen --public` (with no `--file`) gives you the public key of the selected key file.
4. Never run an operator-only command (`secrets setup` except `setup --print`, `set`, `edit`, `unset`, `keygen` except `keygen --public` with no `--file`, `recipient add|rm`, `store init`, `store fix-config`, `store remove`, `push`, `login`, `logout`, `identity add|use|rm`, `project identity <name>`, `project identity --clear`, `project forget`, `mv`, `env <name> <envs>`, `tag add|rm|edit|rename`, `vault tag add|rm`, `expiry set|clear`, `refs fix` except `refs fix --dry-run`, `vault mv|rm`, `pull --force`). The one exception: your prompt relays an instruction or an approval of the operator that names that exact command. Never add `--yes` to a push.
5. Never set `SECRETS_GITLEAKS=0`, and never remove, rename, or hide gitleaks on `PATH`. Before a store write that this shard allows, check for an inherited `SECRETS_GITLEAKS=0`. See [[dev-knw-nsec-rules]] § Rule 10.
6. You are a user. The rules prevent accidents. They are not a security boundary.
7. Change `secrets.toml`, `secrets.local.toml`, and the NUU Secrets lines of `.gitignore` only with the commands of the CLI (`init-project`, `bind`, `unbind`, `target set|rm`, `project gitignore`, `project set`, and `refs fix` with the approval of the operator), not by hand. Write a target table or change the store URL only with the ids or the URL that your prompt gives for the operator.
8. Never run `secrets ui`, also when your prompt asks for it. Never open its URL or login link, its page, or its API, and never read a value or a vault text from it. The UI is for the operator. See [[dev-knw-nsec-rules]] § Rule 11.
9. Use the identity that the CLI selects. Do not add `--identity`, and do not set `SECRETS_IDENTITY` or `SOPS_AGE_KEY_FILE`, unless your prompt names the identity for the operator. An identity problem (`LINKED_MAPPING`, `MAPPING_CHANGED`, `OUTSIDE_PROJECT`, `DUPLICATE_PROJECT`, an unknown identity, `PROJECT_INDEX`, the link warning, the parent-identity warning) is a blocker. See [[dev-knw-nsec-rules]] § Rule 12.
10. Moves to `shared`, tag changes, environment sets, and the store migration are operator writes. No agent migrates a store or the secrets of a local repository. A value that another project has, a value in an undeclared environment, a missing value in a declared environment (also a preview value when dev has one), and names of the previous version are blockers. A missing ref with a `moved` hint: run `secrets refs fix --dry-run` and put the plan in a blocker for the approval of the operator. A `removed` ref is a blocker for the decision of the operator. An expiry warning (`due` or `expired`) goes into your result; never run `expiry set|clear` yourself. See [[dev-knw-nsec-rules]] § Rule 13.
11. Never read or write a vault text. Never run `secrets vault show` (also `--copy`), `secrets vault edit`, or `secrets vault set`, also when your prompt asks for it. Never read the clipboard. A task that needs a password, a login, or another text from the vault is a blocker: the operator does that step. `secrets vault list` is allowed (names only). `secrets vault mv|rm` only with an approval of the operator that your prompt relays. Never tell the operator to put a value of an environment into the vault, or a vault text into `secrets.toml`. See [[dev-knw-nsec-rules]] § Rule 14.
12. Never read, print, copy, or change the values in the block of `.env.local`, and never open or search that file. Change a value in the store (the operator), never in `.env.local`. Before you run an app, run `secrets pull --check`, and `secrets pull` when the block is not current; then run the normal command. When `secrets pull` stops on a managed name outside the block, put each line number and name in a blocker. Do not delete or change the line without an approval of the operator that your prompt relays. See [[dev-knw-nsec-rules]] § Rule 15.

## What Is Different in a Headless Session

1. **No interactive prompt.** Never start a command that waits for a person: `secrets setup` (a guided setup on a terminal), `secrets set` (a hidden prompt), `secrets edit` (an editor), `secrets login` (a browser), `secrets ui` (a web page for a person; you never run it, Rule 8 above), `secrets vault edit` (an editor; you never run it, Rule 11 above), or `secrets refs fix` without `--yes` (a question on a terminal; with no terminal it refuses and changes nothing). These are operator commands. `secrets setup --print` waits for nothing: use it to find the setup steps that remain, and put the operator commands of each `todo` step in your blocker ([[dev-knw-nsec-setup]]).
2. **A blocker goes to a person, never to a guess.** These are blockers:
   - the shell cannot find `secrets` (`command not found`): the operator installs the CLI with `npm install -g @nuucognition/secrets-cli@0.4.0-beta.0` ([[dev-knw-nsec-cli]] § Install). Do not install it, and do not run it with `npx`;
   - a value is missing (`✘ <n> variable(s) have no value in env <env> …`, or a `doctor` line `… is not in the store`);
   - `secrets pull` stops with `✘ No usable age key: …`, or `doctor` says `no age key file at <path>` or `this machine is not a recipient of the store`;
   - `--offline` stops with `… has no cached clone on this machine` and the task forbids network access;
   - a `doctor` item that only the operator can fix ([[dev-sk-nsec-diagnose]]);
   - a tool needs a file with values that is not the block of `.env.local`;
   - `secrets pull` stops on a managed name outside the block (`✘ <path> sets names that secrets.toml gives to local, outside the block of NUU Secrets:` with `  - line <n>: <VAR>`, `PULL_MANAGED_OUTSIDE`): put each line number and name in the blocker. The operator removes the line, or removes the target `local` of the name. Do not delete the line, and do not change the mapping yourself;
   - `secrets pull` stops on a changed block (`PULL_CHANGED_BLOCK`; `--force` needs the approval of the operator), a broken block (`PULL_BROKEN_BLOCK`), a `.env.local` that Git tracks (`PULL_TRACKED`), or a link (`PULL_LINK`) ([[dev-sk-nsec-run_app]], step 7). A `.env.local` that Git does not ignore (`PULL_NOT_IGNORED`) is not a blocker: run `secrets project gitignore`, then pull again;
   - a push to a platform: only the operator runs `secrets push` ([[dev-sk-nsec-plan_push]]);
   - a read stops with `✘ The store file <file> of store <store> has SOPS metadata that the CLI does not decrypt. SOPS did not run, and nothing was read.` (`UNSAFE_STORE`), or with the same message for a vault file (`UNSAFE_VAULT`): put the exact message and its lines in the blocker. Do not open, edit, or decrypt the file, and do not try another command to get the values ([[dev-sk-nsec-diagnose]], step 4);
   - a store write stops with the secret scan (`SECRET_SCAN_FOUND` or `SECRET_SCAN_FAILED`) or with the store policy (`UNSAFE_STORE`, `UNSAFE_RESULT`, or `UNSAFE_VAULT`: a field name or a path, never a value), or its message has a cleanup line or `Nothing was pushed.` ([[dev-sk-nsec-diagnose]], step 10);
   - the environment of a store write that you may run has `SECRETS_GITLEAKS=0` (the `doctor` line `SECRETS_GITLEAKS=0: the CLI does not scan store commits`);
   - an identity problem: a command or `doctor` reports `LINKED_MAPPING`, `MAPPING_CHANGED`, `OUTSIDE_PROJECT`, `DUPLICATE_PROJECT`, `UNKNOWN_IDENTITY` (`✘ There is no identity named …`), `PROJECT_INDEX`, the warning `… has no identity of its own in projects.toml, so the CLI uses …`, or the warning that the identity of a link target does not apply. Put the exact message and the remedy lines of the CLI in the blocker ([[dev-knw-nsec-rules]] § Rule 12);
   - the task asks you to use `secrets ui`, or to read a value from it: report that the UI is for the operator, and give the request of [[dev-sk-nsec-request_value]];
   - a missing ref says `moved to secret://<new name> …`: put the plan of `secrets refs fix --dry-run` in the blocker, and ask for the approval of the fix ([[dev-sk-nsec-diagnose]], step 7). Do not edit or rebind the ref;
   - a missing ref says `removed on …`: the operator decides (bind another name, remove the entry, or write a new value);
   - the task needs a text from the vault (a password, a login, recovery codes, a private key), or asks you to run `secrets vault show|edit|set`: report that the operator does that step ([[dev-sk-nsec-request_value]], step 5);
   - a project needs a value that another project has: the operator decides and runs the move to `shared` (`secrets mv <project>/<VARIABLE> shared/<VARIABLE>`). Do not write a copy, and do not link to the other project;
   - a warning says that a value is not used (`… does not declare env <env> …`): put the variable, the name, the environments, and the two commands of the warning in the blocker;
   - `doctor` warns for names of the previous version, or the task asks you to migrate a store or the secrets of a repository: the migration is an operator write. Use [[dev-sk-nsec-migrate_v2]]: only its read-only steps, and give the operator each write command ([[dev-knw-nsec-migrate_v2]]);
   - a change of a name, a move to `shared`, a tag, an environment set, an expiry date, or a vault entry (`mv`, `tag add|rm|edit|rename`, `vault tag add|rm`, `env <name> <envs>`, `expiry set|clear`, `vault mv|rm`) that your prompt does not relay as an approval of the operator.

   An expiry warning (`secrets: warning: … expires on …` or `… expired on …`) is not a blocker: the command continued. Put each warning (the variable, the secret name, the environment, and the date) in your result.

   For a blocker, do not invent a value or a text. Do not use a placeholder that looks real. Do not remove the variable from the mapping to make the run pass. Do not skip the check and report success. Do not turn off the secret scan. Write the request with [[dev-sk-nsec-request_value]] (a value or a key) or with the report of [[dev-sk-nsec-diagnose]] (an operator fix, also a scan refusal), set the `blockers` key, and deliver it as in [Result Delivery](#result-delivery).

   A store write that printed `secrets: gitleaks is not on PATH, …` had no scan. Check its exit status, and report both facts in your result.
3. **Work that does not need the value goes on.** Finish the parts of the task that do not need the missing value. Then return with the blocker.
4. **Background programs.** After `secrets pull`, run a dev server with its normal command in the background of your harness. Stop it before you return. The process of the turn ends with the turn.
5. **A leak is a blocker of the highest priority.** Stop. Do not repeat the value. Report the variable, the secret name, the environment, and the place of the leak. Say that the session transcript holds the value. Follow [[dev-knw-nsec-rules]] § If a value leaks.

## Orbh Interface Schema

Use the session keys so that the dispatcher and the operator see the state without the transcript. Never put a value in a key. Names, environments, and paths are public.

| Key | Value | When |
|---|---|---|
| `phase` | One of the phase values below | When NUU Secrets work is the main work of the turn. Else keep the phase of your main task. |
| `blockers` | The secret names and environments that need the operator, for example `nuu-track/CLERK_SECRET_KEY (dev), shared/SENTRY_DSN (preview)` | When a blocker starts. Set it to `none` when the blocker is gone. |

```sh
flint orbh session set phase secrets-blocked
flint orbh session set blockers "nuu-track/CLERK_SECRET_KEY (dev): no value in the store"
```

### Phase Values

| Phase | Meaning |
|---|---|
| `secrets-setup` | You write `secrets.toml` and `.gitignore` ([[dev-sk-nsec-setup_project]]). |
| `secrets-run` | You run `secrets pull` and then a program ([[dev-sk-nsec-run_app]]). |
| `secrets-diagnose` | You run `secrets doctor` and act on it ([[dev-sk-nsec-diagnose]]). |
| `secrets-plan` | You run `secrets plan` and prepare the commands for the operator ([[dev-sk-nsec-plan_push]]). |
| `secrets-blocked` | You wait for the operator: a value, a key, a push, an identity decision, the approval of a ref fix, a step that needs a vault text, or an operator fix. |

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
| Run an app, a test, or a script with its secrets | [[dev-sk-nsec-run_app]]: `secrets pull --check`, `secrets pull` when the block is not current, then the normal command |
| `secrets pull` stops on a managed name outside the block, or on another refusal | [[dev-sk-nsec-run_app]] (step 7), then return with the blocker |
| A project has no `secrets.toml`, or a variable has no entry | [[dev-sk-nsec-setup_project]] |
| This machine has no key or no store yet (a new team or a new machine) | `secrets setup --print`, then [[dev-knw-nsec-setup]]; return with the operator commands as a blocker |
| `secrets` fails, or access is not correct | [[dev-sk-nsec-diagnose]] |
| A value or a key is missing | [[dev-sk-nsec-request_value]], then return with the blocker |
| A command or `doctor` reports an identity problem | Stop. [[dev-sk-nsec-diagnose]] (step 6), then return with the blocker |
| A missing ref says `moved to …` or `removed on …` | [[dev-sk-nsec-diagnose]] (step 7), then return with the blocker |
| A task needs a password, a login, or another text from the vault | [[dev-sk-nsec-request_value]] (step 5), then return with the blocker |
| Prepare a push to Vercel, Convex, or Railway | [[dev-sk-nsec-plan_push]], then return with the commands for the operator |
| A command stops with `This store has format 1` (`STORE_FORMAT`), or `doctor` shows `fail  store format 1 …` | The store needs the migration (an operator write). Do not migrate. When the task asks for help with the migration: [[dev-sk-nsec-migrate_v2]] (the read-only steps only), then return with the ordered write commands as a blocker. The guide: [[dev-knw-nsec-migrate_v2]] |
