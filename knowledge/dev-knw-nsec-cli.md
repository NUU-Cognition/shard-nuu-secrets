---
description: "The secrets CLI for agents: each command and its options, the output rules, the exit statuses, the messages that agents see, the operator-only commands, and the known limits of the current build"
---

# Knowledge: The `secrets` CLI

The CLI is `secrets`. Its alias `secret` is the short form of `secrets run`: `secret pnpm dev` is `secrets run pnpm dev`. `ndv` links both commands. This file lists each command, its options, and its output. `secrets <command> --help` shows the same options. The words are the words of [[dev-knw-nsec-rules]] § Words. This file describes the CLI at `secrets` canon `10f612f`.

## Commands for agents

### `secret <cmd>` and `secrets run`

```sh
secret [--env <env>] [--offline] [--keep-env] [--mapping <path> | --project <name>] [--store <name>] [--] <cmd> [args…]
```

Runs `<cmd>` with the variables of the `local` target in its process environment. It writes no file with a plaintext value. A refresh or a first clone of the store changes the Git files and the index of the store clone in the secrets home. These files hold ciphertext and public data only.

| Option | Meaning |
|---|---|
| `--env <env>` | The environment: `dev`, `preview`, `prod`, or `deploy`. The default is `dev`. |
| `--offline` | Use the cached store clone. stderr names its revision. No network call. When the store has no cached clone, the command stops (exit 1) and clones nothing. |
| `--keep-env` | Keep the shell value of a variable that the mapping sets. This also applies to an `optional` variable that has no store value. |
| `--mapping <path>` | The `secrets.toml` of the project. |
| `--project <name>` | The `secrets.toml` with this project name in the current Git repository. |
| `--store <name>` | The store. It must be the store of the `secrets.toml`. Usually you do not need it. |

- Put the options first. The first word that is not an option starts the command. `--` also starts it. The CLI gives the rest to the command without a change.
- Use `--mapping` or `--project`, not both.
- When the mapping has a `ref` entry, the CLI first checks the key file. Then it selects the store and, when necessary, clones it.
- Before the command starts, stderr has one line:
  `secret: <project> · env <env> · store <revision> (<date>) · <n> variables`
  When `optional` entries have no value, the line ends with `· <n> optional not set`.
- When a store value replaces a shell variable, stderr names the variable: `secret: the store value replaces the shell value of <VAR> (--keep-env keeps the shell value)`.
- When an `optional` variable has no store value and the shell has a variable of that name, the CLI removes it from the process environment of the command. stderr says: `secret: the store has no value for the optional <VAR>: the shell value is removed (--keep-env keeps it)`. With `--keep-env`, stderr says: `secret: --keep-env keeps the shell value of the optional <VAR> (the store has no value)`.
- A missing required `ref` stops the command, also when the shell has a variable of that name.
- The CLI gives SIGTERM and SIGHUP to the command and waits for it. SIGINT also goes to the command. On a terminal, the terminal sends Ctrl-C to the command, so the CLI does not send it a second time.
- Steps: [[dev-sk-nsec-run_app]].

### `secrets export`

```sh
secrets export --format dotenv|json|shell [--env <env>] [--to <target>] [--offline] [--mapping <path> | --project <name>] [--store <name>]
```

Prints the variables and their **values** to stdout. It writes no file with a plaintext value itself. Its refresh or first clone of the store is the same as for `secret <cmd>`. stderr has one line: `secrets: export <project> · env <env> · to <target> · store <revision> (<date>) · <n> variables`.

| Option | Meaning |
|---|---|
| `--format <format>` | Required. `dotenv`, `json`, or `shell`. |
| `--env <env>` | The environment. The default is `dev`. |
| `--to <target>` | `local` (the default; it adds the entries of `secrets.local.toml`), `vercel`, `convex`, or `railway`. A platform target does not use the local entries in its output. But the CLI still reads and checks `secrets.local.toml`: a file that is not valid stops the export (exit 1). |
| `--offline`, `--mapping`, `--project`, `--store` | As for `secrets run`. `--offline` with no cached clone stops and clones nothing. |

- With `--format dotenv`, when the set has a variable named `__proto__`, stderr has a warning: `secrets: warning: the dotenv npm package loses the variable __proto__ when it reads this text. …`. The output still has the variable.
- An agent uses `secrets export` only under the conditions of [[dev-knw-nsec-rules]] § Conditions for `secrets export`.

### `secrets plan`

```sh
secrets plan <vercel|convex|railway> --env dev|preview|prod [--json] [--offline] [--mapping <path> | --project <name>] [--store <name>]
```

Shows what `secrets push` sends to a platform: the names and their sources, never a value. It calls no platform. The exit status is 1 when the plan has an error.

| Option | Meaning |
|---|---|
| `<target>` | `vercel`, `convex`, or `railway`. The mapping needs the `[targets.<target>]` table. |
| `--env <env>` | Required. `dev`, `preview`, or `prod`. `--env deploy` is refused: `deploy.enc.yaml` holds the credentials. |
| `--json` | One JSON object with names, statuses, and all error texts. It contains no values. |
| `--offline` | Use the cached store clone. No network call, and no clone when there is none. |
| `--mapping <path>`, `--project <name>`, `--store <name>` | As for `secrets run`. |

- `plan` reads the entries of the target (each `[env]` entry whose `to` has the target), with their values of the environment, and the credentials in `deploy.enc.yaml`. It never uses `secrets.local.toml` for a platform.
- `plan` decrypts the store values and the credentials **in memory**: it needs a key in group `deploy`. With a key that is not in group `deploy`, each credential is `no access`, and an `error` line says `this machine cannot open deploy.enc.yaml: …`. A decrypt error gives the same status, so `no access` alone does not prove the membership. It is not a permission to add access.
- Without `--offline`, `plan` refreshes the store clone (fetch and reset), as `secret <cmd>` does. Follow the refresh rule of [[dev-knw-nsec-rules]] § Rule 6, or use `--offline`.
- A successful plan does not prove that a push passes. `plan` cannot see the state of the platform: the Vercel variables of other environments, the Railway reference variables, or an identifier collision ([Known limits](#known-limits), items 9 and 10).
- Steps: [[dev-sk-nsec-plan_push]].

The text form:

```
store    <store> @ <revision> (<date>)
target   vercel · project <project> · env dev → development
key      deploy.enc.yaml → vercel/token    ok
key      deploy.enc.yaml → vercel/team-id  ok (optional)
set      API_KEY     ← app/api-key
set      LOG_LEVEL   ← literal
remove   OPT_SECRET  ← app/none (empty, optional)
3 variables · 0 errors
```

- `set` sends the value of a `ref`, or a literal. `remove` removes an `optional` entry that has no value or an empty value in the environment. On a Railway `remove` line, the plan adds `Railway can start a deployment for a removal`.
- The status of a credential (`key` line) is `ok`, `missing`, `empty`, `not valid`, `no access`, or `not set` (an optional credential with no value).
- An `error` line: a required `ref` with no value or with an empty value, an empty literal, no access to `deploy.enc.yaml`, or a Railway environment with no entry (`targets.railway.environments has no entry for env <env>: …`). A credential that is `missing`, `empty`, `not valid`, or `no access` is also an error.
- The last line counts the variables and the errors.
- Text output shows the credential status. It omits the detailed credential error for `missing`, `empty`, and `not valid`. Use `--json` for that error text.

The credentials (in `deploy.enc.yaml`, environment `deploy`, group `deploy`):

| Target | Credential | Note |
|---|---|---|
| `vercel` | `vercel/token` | Required. A Vercel access token. |
| `vercel` | `vercel/team-id` | Optional. The team id `team_…`. It is not secret. Without it, the requests go to the personal account of the token. |
| `convex` | `convex/<deployment>/<env>` | Required. A **typed** deploy key of that deployment: `prod:…` for `prod`, `dev:…` for `dev` and `preview`. A key with no type, a preview deploy key, and a project token are refused. |
| `railway` | `railway/token` | Required. A Railway account token or workspace token, not a project token. |

The operator writes each credential with `secrets set <name> --env deploy`. Agents never write a credential.

The environments: Vercel `dev → development`, `preview → preview`, `prod → production`. Convex: the deploy key `convex/<deployment>/<env>` selects the deployment. Railway: the `environments` table of `[targets.railway]`; an environment with no entry is an error.

### `secrets doctor`

```sh
secrets doctor [--mapping <path> | --project <name>] [--store <name>] [--env <env>]… [--online]
```

Checks the tools, the key, the stores, the mapping, the project hygiene, and the disk encryption. Each check is `ok`, `warn`, `fail`, or `skipped`. A `warn` or `fail` check can have a remedy line (`→ …`). The last line is `doctor: <n> ok, <n> warn, <n> fail, <n> skipped`. The exit status is 1 when a check fails.

| Option | Meaning |
|---|---|
| `--mapping <path>`, `--project <name>` | The `secrets.toml` to check. |
| `--store <name>` | Check this store. |
| `--env <env>` | An environment for the `ref` checks. Repeat it for more environments. The default is `dev`, `preview`, and `prod`. |
| `--online` | First refresh each checked clone (fetch and reset to the remote), then check. |

- **Without `--online`**, `doctor` reads the cached clones. It makes no network call and changes nothing. Each store section names the cached revision and its date: `ok  cached revision <revision> (<date>): doctor reads the clone offline, so it can be older than the remote (…)`.
- **With `--online`**, `doctor` fetches each checked clone and resets it to the remote. The store clone is a cache that the CLI owns: the reset discards the local changes and commits of an interrupted command. `--online` never changes the remote, and it never clones a store that has no clone. The line is `ok  revision <revision> (<date>), refreshed from the remote just now (--online)`. A failed refresh is `fail  --online: cannot refresh the clone: …`, and then the checks of the cached clone follow (`warn  cached revision …: the refresh failed, so the clone can be older than the remote`).
- `doctor --online` does not show the changes of `recipients.toml`. `secrets store sync <name>` shows them, with the ids, groups, fingerprints, and commit authors.
- Before `secrets doctor --online`, run plain `secrets doctor` with the same mapping, store, and environment options. Before `secrets store sync <name>`, run plain `secrets doctor` that checks the named store. For sync, pass that store name as the argument. Refresh only stores that the plain check covered ([[dev-knw-nsec-rules]] § Rule 6).
- Steps: [[dev-sk-nsec-diagnose]].

### `secrets whoami`

Shows the local identity: the recipient id, the fingerprint of the key of this machine (or "no key"), each store with the groups of this machine, and the optional NUU account. Without a login, it says "Not logged in" and ``Run `secrets login` to authenticate.`` and exits 0. No other command needs a login, and `login` is an operator command.

### `secrets keygen --public`

```sh
secrets keygen --public
```

Prints the full age public key of the key file of this machine as one line on stdout. It makes no key. It never prints the private key. The CLI reads the key file for you: you never open the key file yourself.

- When the key file has more than one identity, stdout has the first one, and stderr says: `secrets: <file> has <n> identities. This is the first one: the CLI uses it as the key of this machine.`
- A missing, unreadable, or non-regular key file gives `✘ No usable age key: …` (exit 1).
- An invalid identity file can give `✘ age-keygen cannot read the key file <path> (exit <n>).` or `✘ The key file <path> holds no age identity.` Both exit 1. Give the error to the operator. Do not open the file or run `age-keygen -y` yourself.
- Only this form is for agents. `secrets keygen` with no option, `--out <file>`, and `--public --file <path>` are operator commands.

### `secrets store list`, `secrets store sync`, `secrets store clone`

| Command | What it does |
|---|---|
| `secrets store list` | Lists the store clones of this machine: `NAME`, `REVISION`, `URL`. No values. |
| `secrets store sync [<name>]` | Refreshes one store clone, or all. It shows each change of `recipients.toml` since the last sync, with the commit author. It resets the current branch and the work tree of the clone to the remote. An agent always names the store, after a plain `secrets doctor` of that store ([[dev-knw-nsec-rules]] § Rule 6). |
| `secrets store clone <url> [--name <name>]` | Clones an existing store. `secret` without `--offline` clones the store of a mapping when necessary, so you seldom need this command. |

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
| `--optional` | When the store has no value for the name, the CLI does not set the variable and does not stop. `run` then also removes a shell variable of the same name (unless `--keep-env`). Only with `<name>`. A `bind` without `--optional` makes the entry required, also when it was optional before. |
| `--mapping <path>` | The `secrets.toml` to change. The default is the nearest one. |

- `init-project` writes `secrets.toml` in the current folder. It refuses when the file exists. stdout: `created  <path>  project <name>  store <url>`.
- `bind` adds or replaces one entry of `[env]` and keeps every other byte of the file. stdout: `added`, `replaced`, or `unchanged`, then `<VAR> → secret://<name>` (or `→ literal`), the targets, and the path.
- `bind` refuses a platform target that has no `[targets.<target>]` table. Nothing is written.
- `bind --value` refuses a literal that looks like a secret. It gives the variable name to the check. See [Known limits](#known-limits), item 2.
- Steps: [[dev-sk-nsec-setup_project]].

### Help and version

`secrets --help`, `secrets <command> --help`, `secrets help [command]`, and `secrets --version` (`-V`).

## Commands for the operator only

Do not run these commands. The one exception is in [[dev-knw-nsec-rules]] § Rule 5. This table lets you give the operator the correct command.

| Command | Options | What it does |
|---|---|---|
| `secrets set <name>` | `--env <env>` (default `dev`), `--store <name>`, `--mapping <path>`, `--allow-empty` | Writes one value from a hidden prompt or stdin. `--offline` is refused: a write needs the remote. |
| `secrets unset <name>` | `--env <env>` (required), `--store`, `--mapping` | Removes one name from one environment. |
| `secrets edit` | `--env <env>` (required), `--store`, `--mapping` | Opens the values of one store file in `$VISUAL`, `$EDITOR`, or `vi`. The plaintext is in a private temporary file while the editor runs. For `preview`, the editor shows only the preview file, not the dev values under it. |
| `secrets keygen` | none, `--out <file>`, or `--public --file <path>` | Makes the age key of this machine, or (with `--out`) a recovery key file. It refuses to replace an existing key file. `--public --file <path>` prints the public key of another key file. |
| `secrets recipient add <id> <age-public-key>` | `--groups <groups>` (required: `dev`, `prod`, `deploy`), `--store`, `--mapping` | Adds a recipient to groups. |
| `secrets recipient rm <id>` | `--rotate`, `--store`, `--mapping` | Removes a recipient and makes new data keys. |
| `secrets store init <url>` | `--recovery <age-public-key>` (required), `--name`, `--id`, `--key` | Creates a store in an empty Git repository. Its first commit has `recipients.toml`, `.sops.yaml`, `.gitleaks.toml`, `README.md`, and `.gitignore`. |
| `secrets store fix-config` | `--store`, `--mapping` | Writes the generated `.sops.yaml` (from `recipients.toml`) and `.gitleaks.toml` when one is missing or changed: one commit and one push. stdout: `fixed  <file> of store <name> @ <revision>` for each file, or `unchanged  …`. It refuses a store file that is a symbolic link: the operator first removes the link in a checkout of the store (`git rm`, commit, push). |
| `secrets store remove <name>` | none | Removes the clone folder of a store and its entry in `stores.toml` (local state). See below. |
| `secrets login`, `secrets logout` | `login --verbose` | The optional NUU account. |
| `secrets push <target> --env <env>` | `--yes` (required for `--env prod`), `--mapping`, `--project`, `--store` | Sends the mapped variables of one environment to a platform. Every environment is an operator push. See below. |

A write (`set`, `unset`, `edit`, `recipient add|rm`, `store fix-config`) locks the clone and applies the change to the newest remote state. When the store changes, it makes one commit and pushes it, and it reports success only after the remote accepts the push. When nothing changes (for example `unchanged`), it makes no commit.

### `secrets store remove <name>` (operator)

- It makes no network call, and it does not change the remote. stdout: `removed  <name>  <path>`. stderr: ``secrets: the remote is not changed. To use the store again, run `secrets store clone <url>`.``
- The operator runs it from a folder outside the clone. From the clone or a folder below it, it stops: `✘ Cannot remove store <name>: the working folder <path> is inside the clone. Nothing was removed.`
- It refuses local work with exit 1: `✘ Cannot remove store <name>: <reason>. Nothing was removed.` Its commit checks use cached remote-tracking refs. They make no network call.
- Current-branch commits absent from those refs: the operator decides whether to keep them. To discard them, the operator runs `secrets store sync <name>`.
- Another local branch with such commits: the operator runs `git -C "<clone path>" branch -vv`. To discard a branch, the operator runs `git -C "<clone path>" branch -D <branch>`. Sync does not remove it.
- A stash: the operator runs `git -C "<clone path>" stash list`. To discard one entry, the operator runs `git -C "<clone path>" stash drop`. Sync does not remove it.
- Detached HEAD with commits absent from the cached refs: the operator runs `git -C "<clone path>" status`. The operator keeps those commits on a branch and pushes them, or discards them by hand. Sync cannot repair detached HEAD.
- Changed or untracked files: to discard them, the operator runs `secrets store sync <name>`.
- A live lock or unfinished Git operation: give the operator the exact refusal and remedy. Do not reset the clone or delete the lock yourself.
- After the operator resolves the refusal, the operator runs `secrets store remove <name>` again from outside the clone.
- An entry whose clone folder is missing loses only its entry: `removed  <name>  entry only: the clone folder <path> was missing`.
- `✘ The clone folder of store <name> changed during the removal. Nothing was removed.` means that a check found a changed folder.
- After the removal, when the old working folder does not exist any more, stderr says `secrets: the old working folder <path> does not exist any more. The removal is done.` (exit 0).
- A removal that fails late can leave a folder `.removing-<name>-<random>` in the stores folder. This folder is not routine cleanup. Agents do not delete it: report it to the operator.

### `secrets push <target>` (operator)

```sh
secrets push <vercel|convex|railway> --env dev|preview|prod [--yes] [--mapping <path> | --project <name>] [--store <name>]
```

Sends the mapped variables of one environment to a platform. It is an operator command for every environment ([[dev-knw-nsec-rules]] § Rule 9). An agent gives the operator the exact `plan` and `push` commands, and never adds `--yes`.

- `--env prod` needs `--yes`: each push to production needs the approval of the operator. Without `--yes`, the push stops before it reads anything (exit 2): `✘ secrets push <target> --env prod changes production. Each push to production needs the approval of the operator. Nothing was read or sent.` There is no prompt.
- `push` makes the plan first. When the plan has an error, it sends nothing (exit 1): `✘ secrets push <target> --env <env>: the plan has <n> error(s). Nothing was sent.`
- `push` always refreshes the store clone. It has no `--offline`.
- `push` changes only the variables of the plan. It never changes or removes another variable of the platform. A Railway reference variable (`${{…}}`) is never overwritten or removed.
- The output has one line for each variable, then a summary `push <target> · env <env>: …`. No line shows a value or a token:
  - `set <VAR>` and `removed <VAR>`: the platform confirmed the write;
  - `unchanged <VAR>`: the platform has the value already, or has no variable to remove;
  - `skipped <VAR> (reference variable)`: a Railway reference variable; this is not a failure, but the store value is not on the platform;
  - `failed <VAR> (<reason>)`: the platform refused the write, or the push did not send it;
  - `uncertain <VAR> (<reason>; the platform can have the change)`: the request was sent, but the answer did not confirm it.

  A Vercel answer with an unexpected type or an invalid failure list is `uncertain`. A Railway mutation with GraphQL errors is also `uncertain`, except for a confirmed rejection. A confirmed rejection requires no mutation result. Every error must have one of these codes: `GRAPHQL_PARSE_FAILED`, `GRAPHQL_VALIDATION_FAILED`, or `BAD_USER_INPUT`. An error beside a mutation result is `uncertain`.

- After the first failed or uncertain write, the push sends nothing more. The exit status is 0 only when each write is confirmed. stderr names the confirmed writes (`secrets: confirmed writes: …`) and the uncertain writes (`secrets: uncertain writes: …: the platform can have the change.`). With an uncertain write, stderr says `secrets: Check each uncertain variable on the platform before you run the push again.`
- An error shows a fixed text and the HTTP status, for example `an error that the CLI does not know (HTTP <status>)`. The CLI never shows a message, a code, a request id, the body, or a header of an answer.
- An identifier from an answer (a Vercel variable id, a Railway project, service, or environment id, the Convex deployment URL) must hold no credential and no store value of the push, of any length. Else the push stops before that request: `… from the platform holds the credential <name>`, or `… from the platform holds the value of <VAR> (a value that is not secret can be a literal in secrets.toml)`. A literal of `secrets.toml` is public, so it is not checked.

After the push, the effect on each platform:

| Target | Effect |
|---|---|
| `vercel` | Only a new deployment reads the new values of `preview` and `production`. The push prints a `next` line with the deploy command (`vercel deploy --prod` or `vercel deploy`). `development` needs no deployment. `push` does not deploy, and an agent does not deploy. `vercel env pull` writes a file with values: it is never an agent step. |
| `vercel` | A variable of the same name that covers the environment and also another environment or a custom environment is shared: the push sends nothing and fails. The operator changes it in Vercel. A new production variable has the type `sensitive`. |
| `convex` | The deployment reads the new values at once. No deployment is necessary. One request carries all changes; any end other than a complete HTTP 200 makes the whole batch uncertain. |
| `railway` | Set variables are saved with `skipDeploys` (no deployment). A removal is one request, and `Railway can start a deployment for a removal`. The `next` line asks for a deployment of the service with the new values. |

## Output rules

- Normal output never shows a value. The one exception is the stdout of `secrets export`. `secrets push` sends values to a platform and prints names only.
- Results go to stdout: tables, `added …` lines, the `doctor` report, the `plan` lines, the `push` lines, the public key of `keygen --public`.
- Diagnostics go to stderr: the `secret:` line, notices that start with `secrets:`, and errors. A CLI error starts with `✘`. It can have remedy lines, but not always. An error of the option parser starts with `error:`, for example `error: unknown option '--bad'` or `error: required option '--format <format>' not specified`.
- Output names only public facts: variable names, secret names, store names, project names, recipient ids, public keys and key fingerprints, file names, paths, and line numbers.
- The CLI never shows the content of a store file, `.sops.yaml`, `.gitleaks.toml`, `recipients.toml`, `secrets.toml`, `secrets.local.toml`, `stores.toml`, a `.env` file, or the target of a symbolic link. SOPS and Git errors are a fixed summary. A URL is shown without its user information.

## Exit statuses

| Status | Meaning |
|---|---|
| `0` | Success. |
| `1` | An error. This includes the errors of the option parser: an unknown option, a missing required option, or an unknown command. `secrets doctor` exits 1 when a check fails. `secrets plan` exits 1 when the plan has an error. `secrets push` exits 1 for a plan error, a failed write, or an uncertain write. |
| `2` | With `✘ No command to run.`: `secret` or `secrets run` with no command. The program did not start. With `✘ secrets push <target> --env prod changes production. …`: `secrets push --env prod` without `--yes`. Nothing was read or sent. |
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
| `✘ No usable age key: no age key file at <path>.` (or `the age key file <path> is not a regular file.`, or `the age key file <path> cannot be read.`) with ``Run `secrets keygen` (or set SOPS_AGE_KEY_FILE).`` | The CLI found no usable key file before any decrypt. Nothing ran. | Tell the operator the path. Do not make a key. Use the key request of [[dev-sk-nsec-request_value]]. |
| `✘ sops decrypt failed (exit <n>) on <file>: the age key of this machine cannot decrypt the data key of the file.` | SOPS did not decrypt the file with the available key sources. This error does not prove that the selected key file is valid. The CLI does not show the cause. | Run `secrets doctor`. Do not run `sops`. |
| `✘ --offline: the store of <project> (<url>) has no cached clone on this machine. Nothing was cloned.` | `--offline` found no clone. Nothing ran. No network call. | When the task allows network access, run once without `--offline`, or `secrets store clone <url>`. Else report a blocker. |
| `secret: the store has no value for the optional <VAR>: the shell value is removed (--keep-env keeps it)` | The command runs without that variable. | Normal. Use `--keep-env` only when the task needs the shell value. |
| `✘ The stores folder <path> is a symbolic link. The CLI follows no symbolic link below the secrets home. Nothing was changed.` | `stores/` in the secrets home is a link. | Report the path to the operator. Do not move or link folders. |
| `✘ Store <name> has a file that is not a regular file. Nothing was read.` (or `written`, `cloned`) with lines `- <file> is a symbolic link`, `- <file> is committed as a symbolic link (mode 120000)`, `- <file> is committed as a folder`, or `- <file> is not a regular file` | A store file is a link or a special file. The CLI reads and writes no store file through a link. | Report the lines to the operator. The repair is an operator task. |
| `secrets: warning: the dotenv npm package loses the variable __proto__ …` | `export --format dotenv` has a variable named `__proto__`. | Do not use the dotenv format for it. |
| `✘ <path> is not valid:` with `- targets.<target>: …` or `- env.<VAR>.to: "convex" is not allowed: the Convex tool owns <VAR>, …` | A `[targets.*]` table or a Convex target is not valid. Every command of the project stops, also `secret <cmd>`, `run`, and `export`. | Run `secrets doctor`. Fix the table only when the operator or the task gives its content (Rule 8). Else report it. |
| `✘ <path> has no [targets.<target>] table.` | `plan` or `push` for a target with no table. | Report it. Do not add the table on your own (Rule 8). |
| `✘ --env deploy: plan and push deliver dev, preview, prod.` | `deploy.enc.yaml` holds the credentials and is never delivered. | Use `dev`, `preview`, or `prod`. |
| `error    this machine cannot open deploy.enc.yaml: …` (a `plan` line) | The CLI cannot read `deploy.enc.yaml`. The reason can be absent deploy membership or a decrypt error. The credentials show `no access`. | Report the fixed reason. Do not infer membership from this status alone. It is not permission to add access. |
| `✘ secrets push <target> --env prod changes production. Each push to production needs the approval of the operator. Nothing was read or sent.` | `push --env prod` with no `--yes` (exit 2). | Agents do not run `push`. Give the operator the `plan` and `push` commands ([[dev-sk-nsec-plan_push]]). |
| `✘ secrets push <target> --env <env>: the plan has <n> error(s). Nothing was sent.` | The plan has errors (exit 1). | Run `secrets plan <target> --env <env>` and act on each error. |
| `secrets: Check each uncertain variable on the platform before you run the push again.` | A push has uncertain writes (exit 1). | Never run the push again. Show the confirmed and the uncertain writes to the operator. |
| `… from the platform holds the value of <VAR> (a value that is not secret can be a literal in secrets.toml)` or `… holds the credential <name>` | An id of the platform holds a store value or a credential. The push stopped before that request. | The operator decides. Never move a secret value into `secrets.toml`. |
| `✘ There is no secrets.toml in this folder or a parent folder.` | No mapping. | Go to the project folder, or use `--mapping` or `--project`. A new project: [[dev-sk-nsec-setup_project]]. |
| `✘ No command to run.` | `secret` with no command (exit 2). | Give the command. |
| `error: unknown option '<option>'` | The option parser refused an option (exit 1). | Fix the option. An option for the program goes after the program name. |
| `✘ The program "<cmd>" is not on PATH.` | Exit 127. | Check the command name and `PATH`. |
| `✘ Cannot start <cmd> (EACCES).` | Exit 126. | The file is not executable. |
| `✘ Unknown environment "<env>".` | A bad `--env`. | Use `dev`, `preview`, `prod`, or `deploy`. |
| `✘ --store <name> does not match the store of <path>` | `--store` names another store than the mapping. | Remove `--store`. |
| `✘ The --value for <VAR> looks like a secret (<reason>).` | `bind --value` got a value that looks like a secret. Nothing was written. | A value goes in the store: ask the operator. Then bind the secret name. |
| `secrets: cloning the store of <project> from <url>` | The CLI makes a store clone in the secrets home (a network access). | Normal on the first run. Check that the secrets home is the intended one ([[dev-knw-nsec-model]] § The store). |

## Known limits

These limits are in the current build (`secrets` canon `10f612f`).

1. **The `dotenv` npm reader loses `__proto__`.** The `dotenv` package loses a variable named `__proto__` when it reads text. `secrets export --format dotenv` warns on stderr and still prints the variable. Do not name a variable `__proto__`. When a program needs one, use `--format json`, `--format shell`, or `secret <cmd>`.
2. **The secret check of literals is not complete.** `bind` and `doctor` find a known key prefix (for example `sk_live_`), a key block, a JWT, a URL with a password, a high-entropy string of 24 or more characters, and a random token of 16 to 23 letters and digits of one letter case. In the tests of the CLI, the check found about 94% of random base36 tokens and 99.5% of random hex tokens of that length. It misses tokens with few changes between letters and digits (for example 15 letters and 1 digit) and mixed-case tokens shorter than 24 characters. A UUID and a 40-character lowercase hex string (the form of a Git commit SHA) pass, except when a word of the variable name is `SECRET`, `SECRETS`, `TOKEN`, `TOKENS`, `PASSWORD`, `PASSWD`, `PRIVATE`, `CREDENTIAL`, `CREDENTIALS`, `KEY`, `KEYS`, `APIKEY`, or `SALT`. The CLI splits the name into words at `_` and at other characters that are not letters or digits. A camelCase name such as `apiToken` is one word, so a UUID passes there. Do not rely on the check: when a value can be secret, it is not a literal.
3. **Turborepo strict env mode.** In strict mode, Turborepo can drop the variables that `secret` gives to `turbo run`. Run `secret` in the app folder, or ask the operator to add the variables to `passThroughEnv` (or `globalPassThroughEnv`) in `turbo.json`. See [[dev-sk-nsec-run_app]].
4. **`edit --env preview` shows only the preview overlay.** The editor does not show the dev values under it. This is an operator command.
5. **A platform export reads `secrets.local.toml`.** `export --to vercel|convex|railway` does not use the local entries, but a `secrets.local.toml` that is not valid stops it (exit 1).
6. **`doctor` does not run `SOPS_AGE_KEY_CMD`.** When `SOPS_AGE_KEY_CMD` gives the key, `doctor` says `warn  this machine as a recipient: unknown (…)` and `skipped  <env>: the access is unknown (…)`. This is not proof that this machine is not a recipient. An exit status 0 of `doctor` also does not prove that the command gives a valid key. A `secret <cmd>` run shows if the key decrypts.
7. **A late failure of `store remove` can leave a `.removing-*` folder.** The folder `.removing-<name>-<random>` in the stores folder holds the old clone. The operator recovers or deletes it by hand. Agents do not delete it.
8. **A refresh can remove a local link before a check.** `doctor` (without `--online`) refuses a store file that is a symbolic link in the work tree of the clone. A command that refreshes first (`secret`, `export`, `plan` without `--offline`, `push`, `store sync`, `doctor --online`) resets the work tree and can then use the regular file of the remote. A later success does not show that the clone had no link. Report each `doctor` line about a link.
9. **A plan cannot see the platform.** `secrets plan` calls no platform. It cannot see the Vercel variables of other environments (a shared variable fails the push), the Railway reference variables (`skipped`), or an identifier collision (item 10). A successful plan does not prove that the push passes.
10. **A short store value can stop a push.** The identifier check finds a credential or a store value of any length inside an id of the platform. A short value, for example `1`, can be inside an ordinary UUID by chance. Then the push stops before its first write. The operator decides. A value that is not secret can become a literal in `secrets.toml`, which is public and is not checked. A short value that is secret stays in the store, also when this rule stops the push. Agents never move a secret value into `secrets.toml`.
11. **The platform APIs are not proven live.** The tests of the CLI use fake platform servers with the documented API forms. The first push to each live platform is the proof. `Railway can start a deployment for a removal`: Railway documents no control that stops it.
