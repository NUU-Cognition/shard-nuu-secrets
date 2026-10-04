---
description: "The secrets CLI for agents: each command and its options, the output rules, the exit statuses, the messages that agents see, the operator-only commands, and the known limits of the current build"
---

# Knowledge: The `secrets` CLI

The CLI is `secrets`. Its alias `secret` is the short form of `secrets run`: `secret pnpm dev` is `secrets run pnpm dev`. `ndv` links both commands. This file lists each command, its options, and its output. `secrets <command> --help` shows the same options. The words are the words of [[dev-knw-nsec-rules]] § Words.

## Commands for agents

### `secret <cmd>` and `secrets run`

```sh
secret [--env <env>] [--offline] [--keep-env] [--mapping <path> | --project <name>] [--store <name>] [--] <cmd> [args…]
```

Runs `<cmd>` with the variables of the `local` target in its process environment. It writes no file with values. It can clone or refresh the encrypted store clone in the secrets home (Git files).

| Option | Meaning |
|---|---|
| `--env <env>` | The environment: `dev`, `preview`, `prod`, or `deploy`. The default is `dev`. |
| `--offline` | With an existing clone: do not refresh it, and use its cached revision. stderr names that revision. With no clone: this build still clones the store, a network access (see [Known limits](#known-limits), item 8). Before an offline run, check with `secrets store list` that the clone exists. |
| `--keep-env` | Keep a shell variable that has the same name as a store variable. |
| `--mapping <path>` | The `secrets.toml` of the project. |
| `--project <name>` | The `secrets.toml` with this project name in the current Git repository. |
| `--store <name>` | The store. It must be the store of the `secrets.toml`. Usually you do not need it. |

- Put the options first. The first word that is not an option starts the command. `--` also starts it. The CLI gives the rest to the command without a change.
- Use `--mapping` or `--project`, not both.
- Before the command starts, stderr has one line:
  `secret: <project> · env <env> · store <revision> (<date>) · <n> variables`
  When `optional` entries have no value, the line ends with `· <n> optional not set`. The CLI does not give these variables. A variable of that name in the shell environment stays, also without `--keep-env` ([Known limits](#known-limits), item 9). The line does not prove that a variable is absent.
- When a store value replaces a shell variable, stderr names the variable: `secret: the store value replaces the shell value of <VAR> (--keep-env keeps the shell value)`.
- The CLI gives SIGTERM and SIGHUP to the command and waits for it. SIGINT also goes to the command.
- Steps: [[dev-sk-nsec-run_app]].

### `secrets export`

```sh
secrets export --format dotenv|json|shell [--env <env>] [--to <target>] [--offline] [--mapping <path> | --project <name>] [--store <name>]
```

Prints the variables and their **values** to stdout. It writes no file with values itself. It can clone or refresh the encrypted store clone. stderr has one line: `secrets: export <project> · env <env> · to <target> · store <revision> (<date>) · <n> variables`.

| Option | Meaning |
|---|---|
| `--format <format>` | Required. `dotenv`, `json`, or `shell`. |
| `--env <env>` | The environment. The default is `dev`. |
| `--to <target>` | `local` (the default; it adds the entries of `secrets.local.toml`), `vercel`, `convex`, or `railway`. A platform target does not use the local entries in its output. But this build still reads and checks `secrets.local.toml`: a file that is not valid stops the export (exit 1). |
| `--offline`, `--mapping`, `--project`, `--store` | As for `secrets run`. |

An agent uses `secrets export` only under the conditions of [[dev-knw-nsec-rules]] § Conditions for `secrets export`.

### `secrets doctor`

```sh
secrets doctor [--mapping <path> | --project <name>] [--store <name>] [--env <env>]…
```

Checks the tools, the key, the stores, the mapping, the project hygiene, and the disk encryption. It reads the cached clones only (no network). It changes nothing. Each check is `ok`, `warn`, `fail`, or `skipped`. A `warn` or `fail` check can have a remedy line (`→ …`). The last line is `doctor: <n> ok, <n> warn, <n> fail, <n> skipped`. The exit status is 1 when a check fails.

| Option | Meaning |
|---|---|
| `--mapping <path>`, `--project <name>` | The `secrets.toml` to check. |
| `--store <name>` | Check this store. |
| `--env <env>` | An environment for the `ref` checks. Repeat it for more environments. The default is `dev`, `preview`, and `prod`. |

Steps: [[dev-sk-nsec-diagnose]].

### `secrets whoami`

Shows the local identity: the recipient id, the fingerprint of the key of this machine (or "no key"), each store with the groups of this machine, and the optional NUU account. Without a login, it says "Not logged in" and exits 0. No other command needs a login.

### `secrets store list`, `secrets store sync`, `secrets store clone`

| Command | What it does |
|---|---|
| `secrets store list` | Lists the store clones of this machine: `NAME`, `REVISION`, `URL`. No values. |
| `secrets store sync [<name>]` | Refreshes one store clone, or all. It shows each change of `recipients.toml` since the last sync, with the commit author. |
| `secrets store clone <url> [--name <name>]` | Clones an existing store. `secret` clones the store of a mapping when necessary, so you seldom need this command. |

### `secrets init-project` and `secrets bind`

```sh
secrets init-project --name <name> --store <url>
secrets bind <VAR> <name> --to <targets> [--optional] [--mapping <path>]
secrets bind <VAR> --value <literal> --to <targets> [--mapping <path>]
```

| Command or option | Meaning |
|---|---|
| `init-project --name <name>` | Required. The project name: a letter or a digit first, then letters, digits, `.`, `_`, or `-`. 100 characters or fewer. |
| `init-project --store <url>` | Required. The Git URL or path of the store. A URL with a credential is refused. |
| `bind <VAR>` | The variable name. |
| `bind <name>` | The secret name, for example `clerk/secret-key`. The form `secret://clerk/secret-key` is also accepted. |
| `--to <targets>` | Required. A comma-separated list: `local`, `vercel`, `convex`, `railway`. |
| `--value <literal>` | A literal that is not secret. Give exactly one of `<name>` or `--value`. |
| `--optional` | When the environment has no value, the CLI does not give the variable and does not stop. A shell variable of that name stays. Only with `<name>`. A `bind` without `--optional` makes the entry required, also when it was optional before. |
| `--mapping <path>` | The `secrets.toml` to change. The default is the nearest one. |

- `init-project` writes `secrets.toml` in the current folder. It refuses when the file exists. stdout: `created  <path>  project <name>  store <url>`.
- `bind` adds or replaces one entry of `[env]` and keeps every other byte of the file. stdout: `added`, `replaced`, or `unchanged`, then `<VAR> → secret://<name>` (or `→ literal`), the targets, and the path.
- `bind` refuses a platform target that has no `[targets.<target>]` table. Nothing is written.
- Steps: [[dev-sk-nsec-setup_project]].

### Help and version

`secrets --help`, `secrets <command> --help`, `secrets help [command]`, and `secrets --version` (`-V`).

## Commands for the operator only

Do not run these commands. The one exception is in [[dev-knw-nsec-rules]] § Rule 5. This table lets you give the operator the correct command.

| Command | Options | What it does |
|---|---|---|
| `secrets set <name>` | `--env <env>` (default `dev`), `--store <name>`, `--mapping <path>`, `--allow-empty` | Writes one value from a hidden prompt or stdin. `--offline` is refused: a write needs the remote. |
| `secrets unset <name>` | `--env <env>` (required), `--store`, `--mapping` | Removes one name from one environment. |
| `secrets edit` | `--env <env>` (required), `--store`, `--mapping` | Opens the values of one store file in `$VISUAL`, `$EDITOR`, or `vi`. For `preview`, the editor shows only the preview file, not the dev values under it. |
| `secrets keygen` | `--out <file>` | Makes the age key of this machine, or (with `--out`) a recovery key file. |
| `secrets recipient add <id> <age-public-key>` | `--groups <groups>` (required: `dev`, `prod`, `deploy`), `--store`, `--mapping` | Adds a recipient to groups. |
| `secrets recipient rm <id>` | `--rotate`, `--store`, `--mapping` | Removes a recipient and makes new data keys. |
| `secrets store init <url>` | `--recovery <age-public-key>` (required), `--name`, `--id`, `--key` | Creates a store in an empty Git repository. |
| `secrets store fix-config` | `--store`, `--mapping` | Writes `.sops.yaml` again from `recipients.toml`. |
| `secrets login`, `secrets logout` | `login --verbose` | The optional NUU account. |

A write (`set`, `unset`, `edit`, `recipient add|rm`, `store fix-config`) locks the clone and applies the change to the newest remote state. When the store changes, it makes one commit and pushes it, and it reports success only after the remote accepts the push. When nothing changes (for example `unchanged`), it makes no commit.

## Output rules

- Normal output never shows a value. The one exception is the stdout of `secrets export`.
- Results go to stdout: tables, `added …` lines, the `doctor` report.
- Diagnostics go to stderr: the `secret:` line, notices that start with `secrets:`, and errors. A CLI error starts with `✘`. It can have remedy lines, but not always. An error of the option parser starts with `error:`, for example `error: unknown option '--bad'` or `error: required option '--format <format>' not specified`.
- Output names only public facts: variable names, secret names, store names, project names, recipient ids, key fingerprints, file names, and line numbers.
- The CLI never shows the content of a store file, `.sops.yaml`, `recipients.toml`, `secrets.toml`, `secrets.local.toml`, `stores.toml`, or a `.env` file. SOPS and Git errors are a fixed summary. A URL is shown without its user information.

## Exit statuses

| Status | Meaning |
|---|---|
| `0` | Success. |
| `1` | An error. This includes the errors of the option parser: an unknown option, a missing required option, or an unknown command. `secrets doctor` exits 1 when a check fails. |
| `2` | With `✘ No command to run.`: `secret` or `secrets run` with no command. The program did not start. |
| `3` | A conflict in a write: another machine changed the same name to a different value. Nothing was written. (Writes are operator commands.) |
| `126` | `secret <cmd>`: the command is not executable. |
| `127` | `secret <cmd>`: the command is not on `PATH`. Any command: a required program (`sops`, `age-keygen`, `git`) is not on `PATH`. |
| `128 + n` | `secret <cmd>`: the signal `n` stopped the command. For example, 143 is SIGTERM. |
| other | `secret <cmd>` exits with the exit status of the command. |

After the command starts, `secret <cmd>` exits with the exit status of the command. That status can also be 1, 2, 3, 126, or 127. Thus read the message (`✘ …` or `error: …`), not only the status.

## Messages that agents see

| Message (start) | Meaning | Action |
|---|---|---|
| `✘ <n> variable(s) have no value in env <env> of store <store>:` with lines `- <VAR> → secret://<name>` | The store has no value for these names in this environment. Nothing ran. | Ask the operator: [[dev-sk-nsec-request_value]]. |
| `✘ There is no secrets.toml in this folder or a parent folder.` | No mapping. | Go to the project folder, or use `--mapping` or `--project`. A new project: [[dev-sk-nsec-setup_project]]. |
| `✘ sops decrypt failed (exit <n>) on <file>: the age key of this machine cannot decrypt the data key of the file.` | This machine cannot open the file. The key file can be missing (see [Known limits](#known-limits)). | Run `secrets doctor`. Do not run `sops`. |
| `✘ No command to run.` | `secret` with no command (exit 2). | Give the command. |
| `error: unknown option '<option>'` | The option parser refused an option (exit 1). | Fix the option. An option for the program goes after the program name. |
| `✘ The program "<cmd>" is not on PATH.` | Exit 127. | Check the command name and `PATH`. |
| `✘ Cannot start <cmd> (EACCES).` | Exit 126. | The file is not executable. |
| `✘ Unknown environment "<env>".` | A bad `--env`. | Use `dev`, `preview`, `prod`, or `deploy`. |
| `✘ --store <name> does not match the store of <path>` | `--store` names another store than the mapping. | Remove `--store`. |
| `✘ The --value for <VAR> looks like a secret` | `bind --value` got a value that looks like a secret. Nothing was written. | A value goes in the store: ask the operator. Then bind the secret name. |
| `secrets: cloning the store of <project> from <url>` | The CLI makes a store clone in the secrets home. | Normal on the first run. Check that the secrets home is the intended one (see [Known limits](#known-limits)). |

## Known limits

These limits are in the current build (`secrets` canon `15c40cf`). They are in the backlog of the mission.

1. **A missing key file gives a misleading error.** When the key file does not exist, `secret <cmd>` (and each command that decrypts) reports "the age key of this machine cannot decrypt the data key of the file". The true cause can be a missing key file. Run `secrets doctor`. Its `Key` section says `no age key file at <path>`. Tell the operator the path. The operator makes the key (`secrets keygen`) or sets `SOPS_AGE_KEY_FILE`. Do not make a key.
2. **The dev build names `secrets-dev login`.** In a dev build (`ndv use secrets dev`), `secrets whoami` can say "Run `secrets-dev login` …". `ndv` links the dev build as `secrets` and `secret`, so `secrets-dev` is not on `PATH`. The command is `secrets login`. A login is optional, and it is an operator command.
3. **There is no `secrets store remove`.** A `secret` run with another secrets home (a different `SECRETS_HOME`, `NUU_HOME`, or build mode) clones the store into that secrets home. stderr then says `secrets: cloning the store of <project> from <url>`. No command removes a clone. Do not delete the folder and do not edit `stores.toml`. Tell the operator the secrets home and the store name.
4. **`export --format dotenv` and `__proto__`.** The `dotenv` npm reader drops a variable named `__proto__`. Do not name a variable `__proto__`.
5. **`doctor` reads only the cached clones.** It does not fetch. For a current result, run `secrets store sync` first.
6. **The secret check of literals is not complete.** `bind` and `doctor` find many values that look like secrets, but not all of them (for example many short tokens of lowercase letters and digits). Do not rely on the check.
7. **Turborepo strict env mode.** In strict mode, Turborepo can drop the variables that `secret` gives to `turbo run`. Run `secret` in the app folder, or ask the operator to add the variables to `passThroughEnv` (or `globalPassThroughEnv`) in `turbo.json`. See [[dev-sk-nsec-run_app]].
8. **`--offline` clones when no clone exists.** `secret --offline` and `secrets export --offline` skip the refresh of an existing clone. When the secrets home has no clone of the store of the mapping, this build still clones it: a network access. stderr says `secrets: cloning the store of <project> from <url>`. Before an offline run, check with `secrets store list` that the clone exists. When no clone exists and the task forbids network access, report a blocker.
9. **An absent optional variable does not remove an inherited variable.** When the store has no value for an `optional` entry, the CLI does not give that variable. But a variable of the same name in the shell environment stays in the process environment of the program, also without `--keep-env`. The `secret:` line does not show it. When the task needs the variable absent, start the program with `env -u <VAR> secret <cmd>`. Do not show its value.
