---
description: "The secrets CLI for agents: each command and its options, the output rules, the exit statuses, the messages that agents see, the operator-only commands, and the known limits of the current build"
---

# Knowledge: The `secrets` CLI

The CLI is `secrets`. Its alias `secret` is the short form of `secrets run`: `secret pnpm dev` is `secrets run pnpm dev`. `ndv` links both commands. This file lists each command, its options, and its output. `secrets <command> --help` shows the same options. The words are the words of [[dev-knw-nsec-rules]] § Words. This file describes the CLI at `secrets` canon `c1f8b20`.

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
- An agent runs plain `doctor` before `doctor --online` or `store sync <name>`, with the same options, and refreshes only the checked store ([[dev-knw-nsec-rules]] § Rule 6).
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

## Output rules

- Normal output never shows a value. The one exception is the stdout of `secrets export`.
- Results go to stdout: tables, `added …` lines, the `doctor` report, the public key of `keygen --public`.
- Diagnostics go to stderr: the `secret:` line, notices that start with `secrets:`, and errors. A CLI error starts with `✘`. It can have remedy lines, but not always. An error of the option parser starts with `error:`, for example `error: unknown option '--bad'` or `error: required option '--format <format>' not specified`.
- Output names only public facts: variable names, secret names, store names, project names, recipient ids, public keys and key fingerprints, file names, paths, and line numbers.
- The CLI never shows the content of a store file, `.sops.yaml`, `.gitleaks.toml`, `recipients.toml`, `secrets.toml`, `secrets.local.toml`, `stores.toml`, a `.env` file, or the target of a symbolic link. SOPS and Git errors are a fixed summary. A URL is shown without its user information.

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
| `✘ No usable age key: no age key file at <path>.` (or `the age key file <path> is not a regular file.`, or `the age key file <path> cannot be read.`) with ``Run `secrets keygen` (or set SOPS_AGE_KEY_FILE).`` | The CLI found no usable key file before any decrypt. Nothing ran. | Tell the operator the path. Do not make a key. Use the key request of [[dev-sk-nsec-request_value]]. |
| `✘ sops decrypt failed (exit <n>) on <file>: the age key of this machine cannot decrypt the data key of the file.` | SOPS did not decrypt the file with the available key sources. This error does not prove that the selected key file is valid. The CLI does not show the cause. | Run `secrets doctor`. Do not run `sops`. |
| `✘ --offline: the store of <project> (<url>) has no cached clone on this machine. Nothing was cloned.` | `--offline` found no clone. Nothing ran. No network call. | When the task allows network access, run once without `--offline`, or `secrets store clone <url>`. Else report a blocker. |
| `secret: the store has no value for the optional <VAR>: the shell value is removed (--keep-env keeps it)` | The command runs without that variable. | Normal. Use `--keep-env` only when the task needs the shell value. |
| `✘ The stores folder <path> is a symbolic link. The CLI follows no symbolic link below the secrets home. Nothing was changed.` | `stores/` in the secrets home is a link. | Report the path to the operator. Do not move or link folders. |
| `✘ Store <name> has a file that is not a regular file. Nothing was read.` (or `written`, `cloned`) with lines `- <file> is a symbolic link`, `- <file> is committed as a symbolic link (mode 120000)`, `- <file> is committed as a folder`, or `- <file> is not a regular file` | A store file is a link or a special file. The CLI reads and writes no store file through a link. | Report the lines to the operator. The repair is an operator task. |
| `secrets: warning: the dotenv npm package loses the variable __proto__ …` | `export --format dotenv` has a variable named `__proto__`. | Do not use the dotenv format for it. |
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

These limits are in the current build (`secrets` canon `c1f8b20`).

1. **The `dotenv` npm reader loses `__proto__`.** The `dotenv` package loses a variable named `__proto__` when it reads text. `secrets export --format dotenv` warns on stderr and still prints the variable. Do not name a variable `__proto__`. When a program needs one, use `--format json`, `--format shell`, or `secret <cmd>`.
2. **The secret check of literals is not complete.** `bind` and `doctor` find a known key prefix (for example `sk_live_`), a key block, a JWT, a URL with a password, a high-entropy string of 24 or more characters, and a random token of 16 to 23 letters and digits of one letter case. In the tests of the CLI, the check found about 94% of random base36 tokens and 99.5% of random hex tokens of that length. It misses tokens with few changes between letters and digits (for example 15 letters and 1 digit) and mixed-case tokens shorter than 24 characters. A UUID and a 40-character lowercase hex string (the form of a Git commit SHA) pass, except when a word of the variable name is `SECRET`, `SECRETS`, `TOKEN`, `TOKENS`, `PASSWORD`, `PASSWD`, `PRIVATE`, `CREDENTIAL`, `CREDENTIALS`, `KEY`, `KEYS`, `APIKEY`, or `SALT`. The CLI splits the name into words at `_` and at other characters that are not letters or digits. A camelCase name such as `apiToken` is one word, so a UUID passes there. Do not rely on the check: when a value can be secret, it is not a literal.
3. **Turborepo strict env mode.** In strict mode, Turborepo can drop the variables that `secret` gives to `turbo run`. Run `secret` in the app folder, or ask the operator to add the variables to `passThroughEnv` (or `globalPassThroughEnv`) in `turbo.json`. See [[dev-sk-nsec-run_app]].
4. **`edit --env preview` shows only the preview overlay.** The editor does not show the dev values under it. This is an operator command.
5. **A platform export reads `secrets.local.toml`.** `export --to vercel|convex|railway` does not use the local entries, but a `secrets.local.toml` that is not valid stops it (exit 1).
6. **`doctor` does not run `SOPS_AGE_KEY_CMD`.** When `SOPS_AGE_KEY_CMD` gives the key, `doctor` says `warn  this machine as a recipient: unknown (…)` and `skipped  <env>: the access is unknown (…)`. This is not proof that this machine is not a recipient. An exit status 0 of `doctor` also does not prove that the command gives a valid key. A `secret <cmd>` run shows if the key decrypts.
7. **A late failure of `store remove` can leave a `.removing-*` folder.** The folder `.removing-<name>-<random>` in the stores folder holds the old clone. The operator recovers or deletes it by hand. Agents do not delete it.
8. **A refresh can remove a local link before a check.** `doctor` (without `--online`) refuses a store file that is a symbolic link in the work tree of the clone. A command that refreshes first (`secret`, `export`, `store sync`, `doctor --online`) resets the work tree and can then use the regular file of the remote. A later success does not show that the clone had no link. Report each `doctor` line about a link.
