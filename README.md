# NUU Secrets

Teaches agents to use NUU Secrets safely: run apps with secrets, map variables, diagnose, and follow the no-value rules (also for the vault).

NUU Secrets keeps the secrets of a team in one encrypted Git store (SOPS + age). Each project maps its environment variables in a committed `secrets.toml`. `secret <cmd>` runs a program with the values in its process environment and writes no file with a plaintext value. This shard gives an agent the rules and the steps to use the tool without a leak. The agent never sees, writes, or moves a value or a vault text. The operator enters the values (with `secrets set`, or in the local web page `secrets ui`), manages the keys and the identities, and owns the names, the collections, the expiry dates, and the vault. The agent never runs `secrets ui` or `secrets vault show|edit|set`, and it uses the identity that the CLI selects. A moved ref goes to `secrets refs fix --dry-run` and a question to the operator.

This folder is the remote source of the shard (shorthand `nsec`, package `@nuucognition/nuu-secrets`): a clone of `NUU-Cognition/shard-nuu-secrets`. The build is in `Shards/NUU Secrets/`.

## The CLI

The agents use the `secrets` CLI. The operator installs it from npm (Node.js 24 or later):

```sh
npm install -g @nuucognition/secrets-cli@0.3.0-beta.0    # the commands secrets and secret
```

SOPS, age, and Git are also necessary. gitleaks is optional. `secrets doctor` checks each tool. See `knowledge/dev-knw-nsec-cli.md` § Install.

Then the operator runs `secrets setup` on a terminal: the guided setup of the key, the store (a new store, or join a store of the team), and the project. Each step is also a plain command. The operator guide is `knowledge/dev-knw-nsec-setup.md`: also the identities, the project index, and `secrets ui`.

## Structure

- `shard.yaml` — the manifest. It depends on `@nuucognition/flint` `^0` (each `0.x` version: the published core `0.3.x` and `0.4.x`). No types, folders, setup, or install files.
- `dev-init-nsec.md` — the interactive init: what NUU Secrets is, the rules in short, and the situations.
- `dev-hinit-nsec.md` — the headless init for Orbh sessions: the same rules, no interactive prompts, and blockers to the dispatcher or the operator.
- `knowledge/dev-knw-nsec-rules.md` — the complete agent rules (the required reading), also Rule 11 (agents never use `secrets ui`), Rule 12 (identities), Rule 13 (moved refs, collections, and expiry dates), and Rule 14 (agents never read or write a vault text).
- `knowledge/dev-knw-nsec-cli.md` — the install, the commands, the options (also `--identity`), the identity and project index commands, `log`, `refs`, `collection list`, `expiry list`, `vault list`, the operator commands (also `mv`, `collection`, `expiry`, `refs fix`, `vault`, and `secrets ui`), the output rules, the exit statuses, the messages, the secret scan of store commits, and the known limits.
- `knowledge/dev-knw-nsec-setup.md` — the setup for an operator: the guided setup `secrets setup` and `setup --print`, a new store, a new machine, a project, identities, the project index, collections, expiry dates, the change log, the vault, `secrets ui`, the store URL forms, and which steps an agent can do.
- `knowledge/dev-knw-nsec-model.md` — the store model, identities and the key selection order, the project index, secret names, collections, expiry dates, the change log and moved refs, the vault, `secrets.toml`, `secrets.local.toml`, literals, and the environment chain.
- `skills/dev-sk-nsec-run_app.md` — run a program with its secrets.
- `skills/dev-sk-nsec-setup_project.md` — write the mapping of a project.
- `skills/dev-sk-nsec-diagnose.md` — act on `secrets doctor`, `whoami`, `identity list`, `project list`, and `store list|sync`; stop on an identity problem; act on a moved or removed ref (step 7).
- `skills/dev-sk-nsec-request_value.md` — ask the operator for a value (with `secrets set` or `secrets ui`), a key, or a step that needs a vault text.
- `skills/dev-sk-nsec-plan_push.md` — run `secrets plan` and give the operator the `plan` and `push` commands.

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

Version 0.8.0 is for npm `@nuucognition/secrets-cli@0.3.0-beta.0`: the code of the machine branch `even-macbook` of `NUU-Cognition/secrets` at `99cb649`, which goes to canon for this version. It is the CLI of Mission 002 with the change log, `secrets mv`, `secrets refs`, the collections, and the expiry dates (`0.2.0-beta.0`), the check of the SOPS metadata before each decrypt (`0.2.0-beta.1`), and the vault (new in `0.3.0-beta.0`). Install it with the exact version (`npm install -g @nuucognition/secrets-cli@0.3.0-beta.0`): an npm tag can give an older version until the operator moves it. The command facts come from the README and `apps/secrets-cli/src/` of `99cb649`. `0.2.0-beta.1` has no vault. The npm version `0.1.0-beta.1` has the code of canon `471e270`: it also has no identities, no project index, and no `secrets ui`. When the CLI changes, check each command, option, and message of the knowledge files and the skills again. The release notes are in `RELEASE.md`.
