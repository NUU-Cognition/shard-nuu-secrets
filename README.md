# NUU Secrets

Teaches agents to use NUU Secrets safely: run apps with secrets, map variables, diagnose, and follow the no-value rules.

NUU Secrets keeps the secrets of a team in one encrypted Git store (SOPS + age). Each project maps its environment variables in a committed `secrets.toml`. `secret <cmd>` runs a program with the values in its process environment and writes no file with a plaintext value. This shard gives an agent the rules and the steps to use the tool without a leak. The agent never sees, writes, or moves a value. The operator enters the values and manages the keys.

This folder is the remote source of the shard (shorthand `nsec`, package `@nuucognition/nuu-secrets`): a clone of `NUU-Cognition/shard-nuu-secrets`. The build is in `Shards/NUU Secrets/`.

## The CLI

The agents use the `secrets` CLI. The operator installs it from npm (Node.js 24 or later):

```sh
npm install -g @nuucognition/secrets-cli@beta    # the commands secrets and secret
```

SOPS, age, and Git are also necessary. gitleaks is optional. `secrets doctor` checks each tool. See `knowledge/dev-knw-nsec-cli.md` § Install.

Then the operator runs `secrets setup` on a terminal: the guided setup of the key, the store (a new store, or join a store of the team), and the project. Each step is also a plain command. The operator guide is `knowledge/dev-knw-nsec-setup.md`.

## Structure

- `shard.yaml` — the manifest. It depends on `@nuucognition/flint` `^0` (each `0.x` version: the published core `0.3.x` and `0.4.x`). No types, folders, setup, or install files.
- `dev-init-nsec.md` — the interactive init: what NUU Secrets is, the rules in short, and the situations.
- `dev-hinit-nsec.md` — the headless init for Orbh sessions: the same rules, no interactive prompts, and blockers to the dispatcher or the operator.
- `knowledge/dev-knw-nsec-rules.md` — the complete agent rules (the required reading).
- `knowledge/dev-knw-nsec-cli.md` — the install, the commands, the options, the output rules, the exit statuses, the messages, the secret scan of store commits, and the known limits.
- `knowledge/dev-knw-nsec-setup.md` — the setup for an operator: the guided setup `secrets setup` and `setup --print`, a new store, a new machine, a project, the store URL forms, and which steps an agent can do.
- `knowledge/dev-knw-nsec-model.md` — the store model, secret names, `secrets.toml`, `secrets.local.toml`, literals, and the environment chain.
- `skills/dev-sk-nsec-run_app.md` — run a program with its secrets.
- `skills/dev-sk-nsec-setup_project.md` — write the mapping of a project.
- `skills/dev-sk-nsec-diagnose.md` — act on `secrets doctor`, `whoami`, and `store list|sync`.
- `skills/dev-sk-nsec-request_value.md` — ask the operator for a value or a key.
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

The command facts come from the `secrets` CLI at canon `471e270` (`secrets --help` and the README of `NUU-Cognition/secrets`, which is also on the npm page). The npm version `0.1.0-beta.1` of `@nuucognition/secrets-cli` has the same code. When the CLI changes, check each command, option, and message of the knowledge files and the skills again.
