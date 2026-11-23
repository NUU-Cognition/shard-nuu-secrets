# NUU Secrets

Teaches agents to use NUU Secrets safely: pull the values of a project into the block of its `.env.local` and run the app, map variables, diagnose, and follow the no-value rules (also for the block and the vault).

NUU Secrets keeps the secrets of a team in one encrypted Git store (SOPS + age). Each project maps its environment variables in a committed `secrets.toml`. `secrets pull` writes the values of a project (`dev` or `preview`) into one block of its `.env.local`, and keeps the other lines of the file. Then the program runs with its normal command. The agent never reads or changes the block: a value changes in the store. This shard gives an agent the rules and the steps to use the tool without a leak. The agent never sees, writes, or moves a value or a vault text. The operator enters the values (with `secrets set`, or in the local web page `secrets ui`), manages the keys and the identities, and owns the names, the moves to `shared`, the tags, the environment sets, the expiry dates, the vault, and the store migration. The agent never runs `secrets ui` or `secrets vault show|edit|set`, and it uses the identity that the CLI selects. A moved ref goes to `secrets refs fix --dry-run` and a question to the operator.

This folder is the remote source of the shard (shorthand `nsec`, package `@nuucognition/nuu-secrets`): a clone of `NUU-Cognition/shard-nuu-secrets`. The build is in `Shards/NUU Secrets/`.

## The CLI

The agents use the `secrets` CLI. The operator installs it from npm (Node.js 24 or later):

```sh
npm install -g @nuucognition/secrets-cli@0.4.0-beta.0    # the command secrets
```

SOPS, age, and Git are also necessary. gitleaks is optional. `secrets doctor` checks each tool. See `knowledge/dev-knw-nsec-cli.md` § Install.

Then the operator runs `secrets setup` on a terminal: the guided setup of the key, the store (a new store, or join a store of the team), and the project. Each step is also a plain command. The operator guide is `knowledge/dev-knw-nsec-setup.md`: also the identities, the project index, and `secrets ui`.

## Structure

- `shard.yaml` — the manifest. It depends on `@nuucognition/flint` `^0` (each `0.x` version: the published core `0.3.x` and `0.4.x`). No types, folders, setup, or install files.
- `dev-init-nsec.md` — the interactive init: what NUU Secrets is, the rules in short, and the situations.
- `dev-hinit-nsec.md` — the headless init for Orbh sessions: the same rules, no interactive prompts, and blockers to the dispatcher or the operator.
- `knowledge/dev-knw-nsec-rules.md` — the complete agent rules (the required reading), also Rule 11 (agents never use `secrets ui`), Rule 12 (identities), Rule 13 (names, moves to shared, tags, environment sets, refs, expiry dates, and the store migration: operator writes), Rule 14 (agents never read or write a vault text), and Rule 15 (`secrets pull` and the block of `.env.local`).
- `knowledge/dev-knw-nsec-cli.md` — the install, `secrets pull` (the block, `--check`, `--all`, `--force`, and the refusals), the commands, the options (also `--identity`), the identity and project index commands, `list`, `env`, `tag list`, `log`, `refs`, `expiry list`, `vault list`, the operator commands (also `mv`, `env <name> <envs>`, `tag`, `vault tag`, `expiry`, `refs fix`, `vault`, and `secrets ui`), the output rules, the exit statuses, the messages, the secret scan of store commits, and the known limits.
- `knowledge/dev-knw-nsec-setup.md` — the setup for an operator: the guided setup `secrets setup` and `setup --print`, a new store, a new machine, a project, identities, the project index, moves to shared, tags, environment sets, expiry dates, the change log, the vault, `secrets ui`, the store URL forms, and which steps an agent can do.
- `knowledge/dev-knw-nsec-model.md` — the store model, identities and the key selection order, the project index, the secret names by store project, `shared`, links and implicit refs, declared environments, tags, expiry dates, the change log and moved refs, the vault, `secrets.toml`, `secrets.local.toml`, literals, the environments (each reads only its own file), and the block of `.env.local`.
- `knowledge/dev-knw-nsec-migrate_v2.md` — the migration guide for a store of the previous version (store format 1) and its projects: who must migrate, what changes, the backup, the CLI update on each machine first, `secrets store migrate --plan` and `--apply <plan>`, the preview values, `secrets refs fix --all`, the old scripts and `.env*` lines, `secrets pull --all`, `doctor` with 0 fail, each other machine, CI, and the rollback. An operator procedure.
- `skills/dev-sk-nsec-run_app.md` — run a program with its secrets: `secrets pull --check`, `secrets pull` when the block is not current, then the normal command.
- `skills/dev-sk-nsec-setup_project.md` — write the mapping of a project.
- `skills/dev-sk-nsec-diagnose.md` — act on `secrets doctor`, `whoami`, `identity list`, `project list`, and `store list|sync`; stop on an identity problem; act on a moved or removed ref (step 7).
- `skills/dev-sk-nsec-request_value.md` — ask the operator for a value (with `secrets set` or `secrets ui`), a key, or a step that needs a vault text.
- `skills/dev-sk-nsec-plan_push.md` — run `secrets plan` and give the operator the `plan` and `push` commands.
- `skills/dev-sk-nsec-migrate_v2.md` — help the operator with the migration to v2: run only the read-only steps (`doctor`, `secrets store migrate --plan`, a read of the plan, `refs fix --all --dry-run`, `pull --check --all`), and give the operator each write command in order.

## Load

```sh
flint shard start nsec      # interactive: init-nsec.md and the rules
flint shard hstart nsec     # headless (Orbh): hinit-nsec.md and the rules
```

## Edit

Edit the files of this folder. Then build the shard again:

```sh
flint shard build nuu-secrets
```

Version 0.9.0 (not released yet) is for npm `@nuucognition/secrets-cli@0.4.0-beta.0` (not published yet; Task 024 publishes it): the code of the machine branch `even-macbook` of `NUU-Cognition/secrets` with `secrets pull` (Notepad 007, Task 020) and with no `run` command and no alias for it (Task 021). `0.4.0-beta.0` also has all of the v2 work of Notepad 008 (read at `6ee9194`, with the three migration fixes of the review of Task 029): the secret names by store project, `shared`, implicit refs, and links (Task 025), the declared environments with no read of dev by preview (Task 026), the tags (Task 027), the store format 2 and its guard (`store.toml`), `secrets store migrate` and `secrets refs fix --all` (Task 029), and the expiry dates that change the same value inside one recipient group (Task 033). The shard text of the v2 work is Tasks 030 and 035. Below it is the CLI of `0.3.0-beta.0` (`99cb649`): the CLI of Mission 002 with the change log, `secrets mv`, `secrets refs`, the groups by provider, and the expiry dates (`0.2.0-beta.0`), the check of the SOPS metadata before each decrypt (`0.2.0-beta.1`), and the vault (`0.3.0-beta.0`). Install it with the exact version (`npm install -g @nuucognition/secrets-cli@0.4.0-beta.0`): an npm tag can give an older version until the operator moves it. The command facts of `secrets pull` come from the README and `apps/secrets-cli/src/pull/` of `even-macbook` at `4ef2cce`. `0.3.0-beta.0` and older have no `secrets pull`. `0.2.0-beta.1` has no vault. The npm version `0.1.0-beta.1` has the code of canon `471e270`: it also has no identities, no project index, and no `secrets ui`. When the CLI changes, check each command, option, and message of the knowledge files and the skills again. The release notes are in `RELEASE.md`.
