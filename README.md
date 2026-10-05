# NUU Secrets

Teaches agents to use NUU Secrets safely: run apps with secrets, map variables, diagnose, and follow the no-value rules.

NUU Secrets keeps the secrets of a team in one encrypted Git store (SOPS + age). Each project maps its environment variables in a committed `secrets.toml`. `secret <cmd>` runs a program with the values in its process environment and writes no file with a plaintext value. This shard gives an agent the rules and the steps to use the tool without a leak. The agent never sees, writes, or moves a value. The operator enters the values and manages the keys.

This folder is the remote source of the shard (shorthand `nsec`, package `@nuucognition/nuu-secrets`): a clone of `NUU-Cognition/shard-nuu-secrets`. The build is in `Shards/NUU Secrets/`.

## Structure

- `shard.yaml` — the manifest. It depends on `@nuucognition/flint` `^0.3`. No types, folders, setup, or install files.
- `dev-init-nsec.md` — the interactive init: what NUU Secrets is, the rules in short, and the situations.
- `dev-hinit-nsec.md` — the headless init for Orbh sessions: the same rules, no interactive prompts, and blockers to the dispatcher or the operator.
- `knowledge/dev-knw-nsec-rules.md` — the complete agent rules (the required reading).
- `knowledge/dev-knw-nsec-cli.md` — the commands, the options, the output rules, the exit statuses, the messages, and the known limits.
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

The command facts come from the `secrets` CLI at canon `10f612f` (`secrets --help` and the README of `NUU-Cognition/secrets`). When the CLI changes, check each command, option, and message of the knowledge files and the skills again.
