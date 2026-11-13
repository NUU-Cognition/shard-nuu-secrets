---
description: "The secrets CLI for agents: the install, each command and its options (also the setup commands: setup --print, vars, target, unbind, bind --local, project; and --identity, identity list, project list, project identity), the operator commands (also identity add|use|rm, project identity <name>, project forget, and secrets ui, which agents never run), the output rules, the exit statuses, the messages that agents see, the secret scan of store commits, and the known limits of the current build"
---

# Knowledge: The `secrets` CLI

The CLI is `secrets`. Its alias `secret` is the short form of `secrets run`: `secret pnpm dev` is `secrets run pnpm dev`. This file lists each command, its options, and its output. `secrets <command> --help` shows the same options. The words are the words of [[dev-knw-nsec-rules]] § Words. This file describes the CLI of Mission 002 (branch `even-macbook`, not yet on canon): the CLI of canon `471e270` (cycle I: the guided setup and the setup commands), with the identities, the project index, and `secrets ui`. The npm version `0.1.0-beta.1` has the code of canon `471e270`: it has no identities, no project index, and no `secrets ui`. The setup of a team, a machine, and a project is in [[dev-knw-nsec-setup]].

## Install

Only the operator installs the CLI. An agent does not install it.

- **From npm.** `npm install -g @nuucognition/secrets-cli@beta`. The package gives the two commands `secrets` and `secret`. It needs Node.js 24 or later. The version on the npm tag `beta` is `0.1.0-beta.1` (the code of canon `471e270`). The older `0.1.0-beta.0` has no setup commands. The identities, the project index, and `secrets ui` come with a later npm version. `secrets --version` shows the version that is installed. On a build without identities, `secrets --help` lists no `identity` command: then the rules of identities do not apply, and the key file is `$SOPS_AGE_KEY_FILE`, else `~/.config/sops/age/keys.txt`.
- **In the NUU Cognition workspace.** `ndv` links the two commands: `ndv use secrets dev` for the dev build, `ndv use secrets npm beta` for the npm package.
- **The other tools.** The package does not include SOPS, age, Git, or gitleaks. `secrets doctor` checks each tool and gives its install command (section Tools).
- **The build mode.** The npm package is the prod build. Its default secrets home is `<NUU home>/secrets/` ([[dev-knw-nsec-model]] § The store).

When the shell cannot find `secrets` or `secret` (`command not found`, exit status 127), stop. Do not install the CLI, and do not run it with `npx`. Ask the operator to install it with the npm command above.

## Commands for agents

### `secret <cmd>` and `secrets run`

```sh
secret [--env <env>] [--offline] [--keep-env] [--mapping <path> | --project <name>] [--store <name>] [--identity <name>] [--] <cmd> [args…]
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
| `--identity <name>` | The identity (a name of `identities.toml`) for this command. It wins over `SECRETS_IDENTITY`, `SOPS_AGE_KEY_FILE`, the project identity, and the default. Use it only when the operator names the identity ([[dev-knw-nsec-rules]] § Rule 12). |

- **`--identity` on other commands.** Each command that decrypts or reads the key has `--identity <name>` with the same meaning: `run`, `export`, `plan`, `push`, `set`, `unset`, `edit`, `recipient add|rm`, `store init|clone`, `doctor`, `setup`, `keygen`, and `whoami`. The same rule applies: only with the identity that the operator names. The key selection is in [[dev-knw-nsec-model]] § Identities and the key selection.
- Put the options first. The first word that is not an option starts the command. `--` also starts it. The CLI gives the rest to the command without a change.
- Use `--mapping` or `--project`, not both.
- When the mapping has a `ref` entry, the CLI first checks the key file. Then it selects the store and, when necessary, clones it.
- Before the command starts, stderr has one line:
  `secret: <project> · env <env> · store <revision> (<date>) · <n> variables`
  When `optional` entries have no value, the line ends with `· <n> optional not set`.
- When a store value replaces a shell variable, stderr names the variable: `secret: the store value replaces the shell value of <VAR> (--keep-env keeps the shell value)`.
- When an `optional` variable has no store value and the shell has a variable of that name, the CLI removes it from the process environment of the command. stderr says: `secret: the store has no value for the optional <VAR>: the shell value is removed (--keep-env keeps it)`. With `--keep-env`, stderr says: `secret: --keep-env keeps the shell value of the optional <VAR> (the store has no value)`.
- A missing required `ref` stops the command, also when the shell has a variable of that name.
- The CLI gives SIGTERM and SIGHUP to the command and waits for it. SIGINT also goes to the command. On a terminal, the terminal sends Ctrl-C to the command, so the CLI does not send it a second time. See [Signals and tool children](#signals-and-tool-children).
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
secrets doctor [--mapping <path> | --project <name>] [--store <name>] [--env <env>]… [--online] [--identity <name>]
```

Checks the tools, the key, the stores, the mapping, the project hygiene, and the disk encryption. Each check is `ok`, `warn`, `fail`, or `skipped`. A `warn` or `fail` check can have a remedy line (`→ …`). The last line is `doctor: <n> ok, <n> warn, <n> fail, <n> skipped`. The exit status is 1 when a check fails.

| Option | Meaning |
|---|---|
| `--mapping <path>`, `--project <name>` | The `secrets.toml` to check. |
| `--store <name>` | Check this store. |
| `--env <env>` | An environment for the `ref` checks. Repeat it for more environments. The default is `dev`, `preview`, and `prod`. |
| `--online` | First refresh each checked clone (fetch and reset to the remote), then check. |
| `--identity <name>` | Check with this identity. Use it only when the operator names the identity ([[dev-knw-nsec-rules]] § Rule 12). |

- **Without `--online`**, `doctor` reads the cached clones. It makes no network call and changes nothing. Each store section names the cached revision and its date: `ok  cached revision <revision> (<date>): doctor reads the clone offline, so it can be older than the remote (…)`.
- **With `--online`**, `doctor` fetches each checked clone and resets it to the remote. The store clone is a cache that the CLI owns: the reset discards the local changes and commits of an interrupted command. `--online` never changes the remote, and it never clones a store that has no clone. The line is `ok  revision <revision> (<date>), refreshed from the remote just now (--online)`. A failed refresh is `fail  --online: cannot refresh the clone: …`, and then the checks of the cached clone follow (`warn  cached revision …: the refresh failed, so the clone can be older than the remote`).
- `doctor --online` does not show the changes of `recipients.toml`. `secrets store sync <name>` shows them, with the ids, groups, fingerprints, and commit authors.
- **The SOPS line** (section Tools). The CLI needs SOPS 3.11 or later. An older SOPS fails as a missing tool does: `fail  sops <version> is too old (the CLI needs 3.11 or later for set --value-stdin and encrypt from stdin)`, with the remedy `→ install or upgrade to SOPS 3.11 or later: …`. When `doctor` cannot read the version of SOPS, the line stays `ok`.
- **The gitleaks lines** (section Tools). gitleaks is optional, so these lines are `ok` or `warn`. The line uses the same `PATH` lookup as the secret scan ([The secret scan of store commits](#the-secret-scan-of-store-commits)):
  - `ok  gitleaks <version>: the CLI scans each store commit with it`. `<version>` can be `(version unknown)`. This line proves only that `gitleaks version` starts. It does not prove that each flag of the scan works. The scan is tested with gitleaks 8.30.1.
  - `warn  gitleaks is not on PATH: the CLI does not scan store commits`, with the remedy `→ install gitleaks, and the CLI scans each store commit with it: pacman -S gitleaks or brew install gitleaks`.
  - `warn  gitleaks does not start: a write that needs a commit stops (SECRET_SCAN_FAILED)`. With `SECRETS_GITLEAKS=0`, this line is only `warn  gitleaks does not start`.
  - With `SECRETS_GITLEAKS=0`, one more line: `warn  SECRETS_GITLEAKS=0: the CLI does not scan store commits`, with the remedy `→ unset SECRETS_GITLEAKS; use SECRETS_GITLEAKS=0 only as a prefix for one command after you confirm a false positive`. This remedy is for the operator ([[dev-knw-nsec-rules]] § Rule 10).
- **The `.gitleaks.toml` lines** (section Store). `doctor` compares the `.gitleaks.toml` of the clone with the text that the CLI generates. It never shows the content.
  - `ok  .gitleaks.toml allows only the recipient key lines of recipients.toml`: the file is the generated text. This line is not a result of a gitleaks scan.
  - `warn  .gitleaks.toml differs from the text that the CLI generates` (also for the text of an earlier version of the CLI), or `warn  .gitleaks.toml is missing`, with the remedy ``→ run `secrets store fix-config --store <name>` (gitleaks then allows only the recipient key lines of recipients.toml)``. `store fix-config` is an operator command ([The config of stores](#the-config-of-stores)).
- **The key selection** (section Key). The first Key line is `ok  key selection: <reason>`, for example `key selection: identity work (the default of identities.toml)` or `key selection: the default key file (no identity)`. It names the step of the selection ([[dev-knw-nsec-model]] § Identities and the key selection). When the selection is not valid (for example an unknown identity name), the line is `fail  There is no identity named "<name>" (<where>)…` with the remedy ``→ run `secrets identity list`, then select an identity that exists (--identity, SECRETS_IDENTITY, `secrets project identity <name>`, or `secrets identity use <name>`)``. A missing key file of an identity is `fail` with the remedy ``→ run `secrets identity list` to check the key file of the identity <name>``.
- **The project index** (section `Projects (projects.toml)`, only when the file exists). `ok  <n> project(s), each path exists`, or a `warn` line for each missing path (`the path of project <name> is missing: <path>`, ``→ when the project is gone, run `secrets project forget <path>` ``) and for each duplicate entry (`the entry <path> of project <name> is for the same secrets.toml as another entry (identity <name>)`, ``→ run `secrets project identity <name> --mapping <path>` (or --clear): it makes the entries one entry``). `doctor` removes nothing. Both remedies are operator commands.
- **Identity problems of the mapping.** When the mapping step stops on an identity problem (`LINKED_MAPPING`, `MAPPING_CHANGED`, `OUTSIDE_PROJECT`, `DUPLICATE_PROJECT`), `doctor` shows the error and its remedy lines. Stop and report them ([[dev-knw-nsec-rules]] § Rule 12).
- Before `secrets doctor --online`, run plain `secrets doctor` with the same mapping, store, and environment options. Before `secrets store sync <name>`, run plain `secrets doctor` that checks the named store. For sync, pass that store name as the argument. Refresh only stores that the plain check covered ([[dev-knw-nsec-rules]] § Rule 6).
- Steps: [[dev-sk-nsec-diagnose]].

### `secrets whoami`

Shows the local identity: the recipient id, the selected identity (`identity <name> (<reason>)`, when an identity is selected), the fingerprint of the selected key (or "no key"), each store with the groups of this machine, and the optional NUU account. When the selection is not valid, the key row says ``the identity selection is not valid (run `secrets identity list`)``. `--identity <name>` shows another identity: use it only when the operator names it. Without a login, it says "Not logged in" and ``Run `secrets login` to authenticate.`` and exits 0. No other command needs a login, and `login` is an operator command.

### `secrets keygen --public`

```sh
secrets keygen --public
```

Prints the full age public key of the selected key file (with an identity: the key file of the identity) as one line on stdout. It makes no key. It never prints the private key. The CLI reads the key file for you: you never open the key file yourself.

- When the key file has more than one identity, stdout has the first one, and stderr says: `secrets: <file> has <n> identities. This is the first one: the CLI uses it as the key of this machine.`
- A missing, unreadable, or non-regular key file gives `✘ No usable age key: …` (exit 1).
- An invalid identity file can give `✘ age-keygen cannot read the key file <path> (exit <n>).` or `✘ The key file <path> holds no age identity.` Both exit 1. Give the error to the operator. Do not open the file or run `age-keygen -y` yourself.
- Only this form is for agents. `secrets keygen` with no option, `--out <file>`, and `--public --file <path>` are operator commands.

### `secrets identity list`

```sh
secrets identity list [--json]
```

Lists the identities of `identities.toml`: a table `NAME`, `DEFAULT`, `PUBLIC KEY`, `KEY FILE`, `STATE`, `STORES`, then one line `active: <reason>  key file <path>` (or `active: no identity (<reason>)  key file <path>`, or `active: none (<error>)`). `STATE` is `ok` or the problem of the key file. `STORES` lists each store clone with the groups of the identity, for example `nuu-secrets (dev)`. With no identity: `No identities (<path> does not exist).` (or `has none`), the `active:` line, and on stderr the `identity add` commands for the operator. `--json` prints one object: `file`, `exists`, `default`, `active`, and `identities`.

- It shows public keys and key file paths, never a private key. Do not open the key files that it names ([[dev-knw-nsec-rules]] § Rule 4).
- `secrets identity add|use|rm` are operator commands ([Commands for the operator only](#commands-for-the-operator-only)).

### `secrets project list` and `secrets project identity`

```sh
secrets project list [--json]
secrets project identity [--mapping <path>]
```

- `project list` shows the project index: a table `NAME`, `STORE`, `IDENTITY`, `SEEN`, `STATE`, `PATH`. The states are `ok`, `missing` (the path does not exist), `linked` (the path has a symbolic link: its identity does not apply), and `duplicate` (another entry is for the same file). stderr adds one line for each kind: ``secrets: <n> path(s) are missing. When a project is gone, run `secrets project forget <path>`.``, the `duplicate` remedy (`secrets project identity <name> --mapping <path>` or `secrets project forget <path>`), and the `linked` remedy (`secrets project identity --clear --mapping <path>`). Each remedy is an operator command. `--json` prints one object.
- `project identity` with no name and no `--clear` shows the identity of the project on this machine: `identity  <name or none>  project <project>  <path>`. With no identity, stderr says `secrets: to set one: secrets project identity <name>` (an operator command). A `secrets.toml` with a link in its path, or outside the project of the current folder, shows `identity  none`. It changes nothing.
- `project identity <name>`, `project identity --clear`, and `project forget` are operator commands.

### `secrets store list`, `secrets store sync`, `secrets store clone`

| Command | What it does |
|---|---|
| `secrets store list` | Lists the store clones of this machine: `NAME`, `REVISION`, `URL`. No values. |
| `secrets store sync [<name>]` | Refreshes one store clone, or all. It shows each change of `recipients.toml` since the last sync, with the commit author. It resets the current branch and the work tree of the clone to the remote. An agent always names the store, after a plain `secrets doctor` of that store ([[dev-knw-nsec-rules]] § Rule 6). |
| `secrets store clone <url> [--name <name>]` | Clones an existing store. `secret` without `--offline` clones the store of a mapping when necessary, so you seldom need this command. When the remote is empty, it stops (exit 1): `✘ The remote <url> is empty.` with the line ``For a new store, run `secrets store init <url> --recovery <age-public-key>`.`` `store init` is an operator command. |

- With no store on this machine and no `secrets.toml` in the folder, a command that needs a store stops (exit 1): `✘ There is no store on this machine and no secrets.toml here.`, then `Next: secrets store clone <url>` and `Or: secrets store init <url> --recovery <age-public-key>`. The same two lines follow `✘ The secrets home <path> does not exist. Nothing was changed.` and `✘ The stores folder <path> does not exist. Nothing was changed.` `secrets store sync` with no store gives `✘ There is no store on this machine.` and `Next: secrets store clone <url>`. Go to the project folder: its `secrets.toml` selects the store. Else ask the operator which store to clone. Only the operator creates a store.
- When the selected branch is missing from the remote, a command that refreshes the clone stops: `✘ The remote of store <name> has no branch "<branch>".` with `Next: secrets store init <url> --recovery <age-public-key>`. `secrets store sync` shows the same fact as `secrets: store <name>: The remote of store <name> has no branch "<branch>".` Report it to the operator.

### `secrets init-project` and `secrets bind`

```sh
secrets init-project --name <name> --store <url> [--no-gitignore]
secrets bind <VAR> <name> --to <targets> [--optional] [--mapping <path>]
secrets bind <VAR> --value <literal> --to <targets> [--mapping <path>]
secrets bind <VAR> <name> --local [--optional] [--mapping <path>]
secrets bind <VAR> --value <literal> --local [--mapping <path>]
```

| Command or option | Meaning |
|---|---|
| `init-project --name <name>` | Required. The project name: a letter or a digit first, then letters, digits, `.`, `_`, or `-`. 100 characters or fewer. |
| `init-project --store <url>` | Required. The Git URL or path of the store. A URL with a credential, or a URL that is not a usual form, is refused ([[dev-knw-nsec-setup]] § Store URL forms). |
| `init-project --no-gitignore` | Do not add the `.gitignore` lines. Use it only when the operator asks. |
| `bind <VAR>` | The variable name. |
| `bind <name>` | The secret name, for example `clerk/secret-key`. The form `secret://clerk/secret-key` is also accepted. |
| `--to <targets>` | Required without `--local`. A comma-separated list: `local`, `vercel`, `convex`, `railway`. |
| `--local` | Write the entry to `secrets.local.toml` beside `secrets.toml` (this machine only). The only target is `local`, and `--to` can be omitted. |
| `--value <literal>` | A literal that is not secret. Give exactly one of `<name>` or `--value`. |
| `--optional` | When the store has no value for the name, the CLI does not set the variable and does not stop. `run` then also removes a shell variable of the same name (unless `--keep-env`). Only with `<name>`. A `bind` without `--optional` makes the entry required, also when it was optional before. |
| `--mapping <path>` | The `secrets.toml` to change. The default is the nearest one. |

- `init-project` writes `secrets.toml` in the current folder. It refuses when the file exists. By default it first adds the missing `.gitignore` lines (as `secrets project gitignore`), and it writes `secrets.toml` only when that step passes. stdout: `created  <path>  project <name>  store <url>`, then `added  .env*, secrets.local.toml  to <path>/.gitignore` (only the missing lines) or `unchanged  <path>/.gitignore  Git ignores .env* and secrets.local.toml`. stderr: `secrets: Next: secrets bind <VAR> <name> --to local`.
- `bind` adds or replaces one entry of `[env]` and keeps every other byte of the file. stdout: `added`, `replaced`, or `unchanged`, then `<VAR> → secret://<name>` (or `→ literal`), the targets, and the path.
- `bind` refuses a platform target that has no `[targets.<target>]` table: `✘ The target <t> has no [targets.<t>] table in <path>. Nothing was written.` with the hint `Add a [targets.<t>] table first: secrets target set <t> …`.
- `bind --value` refuses a literal that looks like a secret. It gives the variable name to the check. See [Known limits](#known-limits), item 2.
- `bind --local` writes only `secrets.local.toml`; `secrets.toml` does not change. It makes the file (mode 600) when it is missing. It refuses (`NOT_IGNORED`, nothing written) when Git does not ignore `secrets.local.toml` (`Next: secrets project gitignore`), when Git tracks it (the hint names `git rm --cached secrets.local.toml`), and when the project folder is not in a Git work tree. `--local` with a target other than `local` is refused (`BAD_TARGET`). stdout: `added|replaced|unchanged  <VAR> → …  to local  in <path of secrets.local.toml>`. When `secrets.toml` also has the variable for `local`, stderr says that the local entry wins on this machine.
- Steps: [[dev-sk-nsec-setup_project]].

### `secrets unbind`

```sh
secrets unbind <VAR> [--local] [--mapping <path>]
```

- It removes the one `[env]` line of `<VAR>` (with its comment) from `secrets.toml`, or with `--local` from `secrets.local.toml`. stdout: `removed  <VAR>  from <path>`.
- A missing entry: `✘ <VAR> is not in [env] of <path>. Nothing was written.` (`NO_ENTRY`, exit 1). With `--local` and no local file: `✘ <path> does not exist. Nothing was written.`
- stderr notes: `secrets: secrets.local.toml still has <VAR> for this machine.` (the project entry is gone, the local entry stays), and `secrets: secrets.toml still has <VAR>; its entry is used for local again.` (after `--local`).
- Do not remove an entry to make a run pass. A missing value goes to the operator ([[dev-sk-nsec-request_value]]).

### `secrets target set` and `secrets target rm`

```sh
secrets target set vercel --project <name-or-id> [--mapping <path>]
secrets target set convex --deployment <short-name> [--mapping <path>]
secrets target set railway --project <name-or-id> --service <name-or-id> --env <store-env>=<railway-env> [--env …] [--mapping <path>]
secrets target rm <vercel|convex|railway> [--mapping <path>]
```

- `target set` adds or replaces one `[targets.<target>]` table with the checks of the target tables ([[dev-knw-nsec-model]] § The target tables). `--env` for Railway is repeatable; the store environments are `dev`, `preview`, `prod`; at least one. stdout: `added|replaced|unchanged  [targets.<t>]  <fields>  in <path>`, for example `added  [targets.vercel]  project web  in <path>`.
- `target set` errors (exit 1, nothing written): `Unknown target "<t>".`; `<--opt> is not an option of the <t> target.`; `The railway target needs at least one --env <store-env>=<railway-env>.`; `The <t> target is not valid. Nothing was written:` with one line for each problem.
- `target rm` removes the table. It refuses while an entry has the target in `to`: `✘ <n> [env] entry has <t> in to: <VAR>, …. Nothing was written.` (`TARGET_IN_USE`) with the hint ``Remove <t> from the to list of each entry first (secrets bind <VAR> <name> --to <targets>), or remove the entry (secrets unbind <VAR>).`` A missing table: `✘ <path> has no [targets.<t>] table. Nothing was written.` (`NO_TARGET`). stdout: `removed  [targets.<t>]  in <path>`.
- A target and its fields are operator decisions ([[dev-knw-nsec-rules]] § Rule 8). Run `target set` only with the ids that the operator or the task gives. Never guess an id.

### `secrets project gitignore` and `secrets project set`

```sh
secrets project gitignore [--mapping <path>]
secrets project set [--store <url>] [--name <name>] [--mapping <path>]
```

- `project gitignore` adds the missing lines `.env*` and `secrets.local.toml` at the end of the `.gitignore` of the project folder, under the comment `# NUU Secrets: files with values, and the mapping of this machine`. Git decides what is missing (`git check-ignore`, as `doctor`). stdout: `added  .env*, secrets.local.toml  to <path>/.gitignore` (only the missing lines) or `unchanged  <path>/.gitignore  Git ignores .env* and secrets.local.toml`. When Git still does not ignore a path after the write, or a Git call fails after the write, the CLI puts the old file back and refuses (`NOT_IGNORED`). An ignore line does not change a file that Git tracks already.
- `project set` changes `name`, `store`, or both in `[project]`. The URL checks are the checks of `init-project`. stdout: `changed|unchanged  [project]  name <name>  store <url>  in <path>`. stderr after a store change: `secrets: the next command that reads the store uses the new URL (it clones the store when this machine has no clone of it).` Change the store only with the URL that the operator gives: it changes where the values of the project come from.

### `secrets vars`

```sh
secrets vars [--unmapped] [--json] [--mapping <path>]
```

- It scans the folder of the mapping for the variable names that the source code reads: `process.env.NAME`, `process.env["NAME"]`, `import.meta.env.NAME`, destructuring of `process.env` and `import.meta.env`, `Bun.env.NAME`, `Deno.env.get("NAME")`, `os.environ["NAME"]`, `os.environ.get("NAME")`, `os.getenv("NAME")`. Files: `.js .jsx .mjs .cjs .ts .tsx .mts .cts .vue .svelte .astro .py`.
- It never reads a `.env*` file or folder. It skips `.git`, `node_modules`, build outputs (`dist`, `build`, `.next`, `.turbo`, `.vercel`, `out`, `coverage`, …), Python environments, a sub-folder with its own `secrets.toml`, symbolic links, binary files, and files over 1 MiB. It shows names and file paths only, never a source line.
- The states: `mapped` (in `secrets.toml`), `local` (only in `secrets.local.toml`), `tool` (a runtime, a platform, or a tool owns it, for example `NODE_ENV`, `CI`, `PORT`, the `VERCEL_*` names, the Convex tool names), `unmapped`.
- stdout: a table `NAME  STATE  FILES`, then `vars: <n> name(s) in <m> file(s): <a> mapped, <b> local, <c> tool, <d> unmapped`. `--unmapped` shows only the `unmapped` rows (the summary counts all). `--json` prints one object with `variables: [{ name, state, files }]` and the counts.
- A `tool` name gets no entry. Some `unmapped` names can be names that the framework sets itself: check the documentation of the app.

### `secrets setup --print`

`secrets setup --print` shows the state of each setup step and the exact commands of each step that remains. It changes nothing, takes no lock, makes no network call, and exits 0. The output form and the use are in [[dev-knw-nsec-setup]] § `secrets setup --print`. `secrets setup` without `--print` is an operator command.

### Help and version

`secrets --help`, `secrets <command> --help`, `secrets help [command]`, and `secrets --version` (`-V`).

## Commands for the operator only

Do not run these commands. The one exception is in [[dev-knw-nsec-rules]] § Rule 5. This table lets you give the operator the correct command.

| Command | Options | What it does |
|---|---|---|
| `secrets setup` | none (`--print` is for agents) | The guided setup on a terminal: tools, key, store (new, join, or on this machine), project, targets, variables, values, check. It makes keys and stores and writes values. See [[dev-knw-nsec-setup]]. |
| `secrets set <name>` | `--env <env>` (default `dev`), `--store <name>`, `--mapping <path>`, `--allow-empty` | Writes one value from a hidden prompt or stdin. `--offline` is refused: a write needs the remote. |
| `secrets unset <name>` | `--env <env>` (required), `--store`, `--mapping` | Removes one name from one environment. |
| `secrets edit` | `--env <env>` (required), `--store`, `--mapping` | Opens the values of one store file in `$VISUAL`, `$EDITOR`, or `vi`. The plaintext is in a private temporary file while the editor runs. For `preview`, the editor shows only the preview file, not the dev values under it. |
| `secrets keygen` | none, `--out <file>`, or `--public --file <path>` | Makes the age key of this machine, or (with `--out`) a recovery key file. It refuses to replace an existing key file. `--public --file <path>` prints the public key of another key file. |
| `secrets recipient add <id> <age-public-key>` | `--groups <groups>` (required: `dev`, `prod`, `deploy`), `--store`, `--mapping` | Adds a recipient to groups. |
| `secrets recipient rm <id>` | `--rotate`, `--store`, `--mapping` | Removes a recipient and makes new data keys. A machine cannot remove its own key: `✘ <id> is the key of this machine. This machine cannot remove its own key.` with the line ``On another machine that is a recipient of the same groups, run `secrets recipient rm <id>`.`` |
| `secrets store init <url>` | `--recovery <age-public-key>` (required), `--name`, `--id`, `--key` | Creates a store in an empty Git repository. Its first commit has `recipients.toml`, `.sops.yaml`, `.gitleaks.toml`, `README.md`, and `.gitignore`. The same scan rules apply to this first commit. |
| `secrets store fix-config` | `--store`, `--mapping` | Writes the generated `.sops.yaml` (from `recipients.toml`) and `.gitleaks.toml` when one is missing or changed, also the `.gitleaks.toml` text of an earlier version: one commit and one push. stdout: `fixed  <file> of store <name> @ <revision>` for each file, or `unchanged  …`. It refuses a store file that is a symbolic link: the operator first removes the link in a checkout of the store (`git rm`, commit, push). Each existing store needs it once after cycle H ([The config of stores](#the-config-of-stores)). |
| `secrets store remove <name>` | none | Removes the clone folder of a store and its entry in `stores.toml` (local state). See below. |
| `secrets login`, `secrets logout` | `login --verbose` | The optional NUU account. |
| `secrets push <target> --env <env>` | `--yes` (required for `--env prod`), `--mapping`, `--project`, `--store` | Sends the mapped variables of one environment to a platform. Every environment is an operator push. See below. |
| `secrets identity add <name>` | `--key <file>` (an existing age key file; without it, a new key in `<secrets home>/keys/<name>.txt`, mode 600), `--default` | Adds an identity. A name has letters, digits, `.`, `_`, and `-`. It refuses an existing name (`IDENTITY_EXISTS`) and a key file with a passphrase. For a new key, stderr gives the admin command `secrets recipient add "<id>" <age-public-key> --groups dev`. |
| `secrets identity use <name>` | none | Makes an identity the default. stdout: `default  <name>  key file <path>`. |
| `secrets identity rm <name>` | none | Removes an identity from `identities.toml`. The key file stays: the CLI never deletes a key. stdout: `removed  <name>  the key file <path> stays`. |
| `secrets project identity <name>` | `--clear`, `--mapping <path>` | Sets (or with `--clear`, removes) the identity of the project on this machine, in `projects.toml`. It refuses a `secrets.toml` that step 4 cannot use (a link in its path, a file outside the project, a file that changed during the command). It changes no project file. |
| `secrets project forget <name\|path>` | none | Removes the entries of a project from `projects.toml`. A word with a `/`, or a word that starts with `.` or `~`, is a path. stdout: `forgot  <name>  <path>` for each entry. It changes no project file. |
| `secrets ui` | `--identity <name>`, `--no-open`, `--idle <minutes>` | The local web page for the operator. **Agents never run it, also when the operator asks** ([[dev-knw-nsec-rules]] § Rule 11). See below. |

A write (`set`, `unset`, `edit`, `recipient add|rm`, `store fix-config`) locks the clone and applies the change to the newest remote state. When the store changes, the CLI applies the scan rules below. It commits the exact bytes that it read and pushes the commit ([The secret scan of store commits](#the-secret-scan-of-store-commits)). It reports success only after the remote accepts the push. When nothing changes (for example `unchanged`), it makes no commit and runs no scan.

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

### `secrets ui` (operator only, never an agent)

```sh
secrets ui [--identity <name>] [--no-open] [--idle <minutes>]
```

- It needs a terminal on stdin and stdout. Without one: `✘ secrets ui needs a terminal on stdin and stdout: the UI is for a person at a terminal. Nothing was started.` (`NO_TERMINAL`, exit 1), with the hints ``Run `secrets ui` in a terminal.`` and ``An agent does not run secrets ui. It uses `secret <cmd>`, and asks the operator to set the values.``
- `--identity <name>` must name an identity of `identities.toml`, else `✘ There is no identity named "<name>" (--identity). Nothing was started.` (`UNKNOWN_IDENTITY`). `--idle` is a whole number of minutes, 1 to 1440 (default 15), else `✘ --idle must be a whole number of minutes (1 to 1440). Nothing was started.` (`BAD_IDLE`).
- The server listens on `127.0.0.1` only, on a random port. The terminal shows the login link `http://127.0.0.1:<port>/#t=<token>` (one use, 60 seconds). The page then asks for an identity. The facts for the operator are in [[dev-knw-nsec-setup]] § The UI: `secrets ui`.
- Each write of the page uses the write path of the CLI, with the same scan rules and messages ([The secret scan of store commits](#the-secret-scan-of-store-commits)). The terminal shows one line for each write and each sync, with names only. Each reveal or copy writes one line in `<secrets home>/ui-reveal.log` (never the value).

### `secrets push <target>` (operator)

```sh
secrets push <vercel|convex|railway> --env dev|preview|prod [--yes] [--mapping <path> | --project <name>] [--store <name>]
```

Sends the mapped variables of one environment to a platform. It is an operator command for every environment ([[dev-knw-nsec-rules]] § Rule 9). An agent gives the operator the exact `plan` and `push` commands, and never adds `--yes`.

- `--env prod` needs `--yes`: each push to production needs the approval of the operator. Without `--yes`, the push stops before it reads anything (exit 2): `✘ secrets push <target> --env prod changes production. Each push to production needs the approval of the operator. Nothing was read or sent.`, then the two lines `Next: secrets plan <target> --env prod` and `With the approval of the operator, run the push again with --yes.` There is no prompt. The second line is for the operator: an agent never adds `--yes`.
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

## The secret scan of store commits

The CLI makes each store commit itself: `secrets store init` and each write that changes the store (`set`, `unset`, `edit`, `recipient add|rm`, `store fix-config`). Before the commit, it scans the exact bytes of the change unless gitleaks is missing or `SECRETS_GITLEAKS=0`. These commands are operator commands. The agent rules are in [[dev-knw-nsec-rules]] § Rule 10.

How the scan works:

- The CLI reads each added or changed file of the clone once. It writes a copy of the bytes to a private temporary folder, with a name that the CLI makes and one last line with a canary (a fake token, random at run time). It runs `gitleaks dir --redact` on the copies with the config text that the CLI generates for `.gitleaks.toml` (one text for the store file and the scan). The CLI writes this text to a private file. The scan does not read the `.gitleaks.toml` of the store, a `.gitleaksignore`, a `gitleaks:allow` comment, or a `GITLEAKS_*` variable. Each copy must give its canary finding, else the scan fails.
- After a clean scan, the CLI stages exactly these bytes and commits them. Before the push, it checks that `HEAD` is one new commit on the base revision, with exactly the scanned files.
- **No Git hook runs in the managed clone.** Each Git call of the CLI has `core.hooksPath=/dev/null`. The CLI removes `GIT_CONFIG_PARAMETERS` from each Git call, and the commit has `--no-verify`. Hooks of a local bare remote can run on the remote side of a push.
- **The switch.** `SECRETS_GITLEAKS=0` (exactly `0`) turns the scan off: no scan and no warning. Any other value keeps the scan on. The store policy check (each value encrypted, the recipients of each file) still runs. Agents never set the switch ([[dev-knw-nsec-rules]] § Rule 10).
- **"Not on PATH".** No `PATH` entry has an executable `gitleaks` when each entry is resolved as the OS resolves it. An empty or relative entry is resolved against the command folder. An unset `PATH` uses the OS default search path: `/bin:/usr/bin` on glibc Linux, `/usr/bin:/bin` on macOS. On musl Linux, the default is `/usr/local/bin:/bin:/usr/bin`. When the Linux C library is unknown, the CLI uses this same path. It can then start gitleaks from `/usr/local/bin` first. There is no exception for any `PATH` form. A `gitleaks` that the CLI finds but that does not start is a failed scan, not a missing tool.
- **Limits.** A changed file with a NUL byte, a changed file of more than 16 MiB, or more than 32 MiB of changed files together is not scanned: the scan fails. The time limit of one scan is 60 s.
- **Manual commits.** The scan covers only the commits of the CLI. It does not cover a manual commit in another checkout of the store. Change the managed store only through `secrets`.

Requirements of the scan:

- Linux or macOS. On Windows, a write that needs a commit stops with `SECRET_SCAN_FAILED` when the scan runs.
- gitleaks 8 (tested with 8.30.1). It is optional: without it, each store write that makes a commit prints one warning and continues.
- Git with `rev-parse --show-object-format`, `read-tree --empty`, `hash-object --no-filters`, and `update-index -z --index-info` (tested with Git 2.55.0). A Git without one of them stops the write.

### The config of stores

Cycle H (canon `35ecaba`) changed the generated `.gitleaks.toml` text. The new text keeps the default rules of gitleaks and has one allowlist for the exact recipient key line `key = "age1…"` of `recipients.toml`. It has no `paths`, so `gitleaks git` and `gitleaks dir` give the same findings. Any other token in `recipients.toml` (a comment, another key, a value) is a finding.

- Each existing store needs one `secrets store fix-config --store <name>`. This is an operator command: one commit and one push. Agents never run it.
- The operator runs it once for each store, on one machine, with the build of canon `35ecaba` or a later build. An older build writes the old text again.
- Until then, `secrets doctor` shows `warn  .gitleaks.toml differs from the text that the CLI generates` with the remedy ``→ run `secrets store fix-config --store <name>` (gitleaks then allows only the recipient key lines of recipients.toml)``. After the repair, `doctor` shows `ok  .gitleaks.toml allows only the recipient key lines of recipients.toml` (on another machine, after its next refresh).
- The secret scan of the CLI does not read the file of the store. Thus the scan works before the repair too.
- An agent that sees the `warn` line reports it to the operator with the exact command `secrets store fix-config --store <name>` ([[dev-sk-nsec-diagnose]], step 4).

### The messages of a store write

The text output does not print the code, and the write commands have no `--json`. Identify the code from the start of the message:

| Code | Message (start) | Meaning |
|---|---|---|
| `SECRET_SCAN_FOUND` | `✘ The secret scan (gitleaks) found <n> possible secret(s) in the change to store <store>. Nothing was committed or pushed.` Then one line for each rule and file: `- <rule> in <file label>`, for example `- github-pat in dev.enc.yaml`. Then two hints: `Check each file above. …` and ``Use `SECRETS_GITLEAKS=0` only as a prefix for one command after you confirm a false positive. Do not export it. The store policy check still runs.`` | The scan found a possible secret in the change. The write stops (exit 1). The second hint is for the operator only. |
| `SECRET_SCAN_FAILED` | `✘ The secret scan (gitleaks) of the change to store <store> did not complete: <reason>. Nothing was committed or pushed.` Then two lines: `Next: secrets doctor` and `Then run the command again. Do not bypass a failed scan.` | The scan did not give a clean result. The write stops (exit 1). The reason is a fixed text, for example `gitleaks failed (exit <n>)`, `gitleaks did not start`, `gitleaks did not finish within 60 s`, `the scan did not read each file to its end`, `gitleaks wrote no report`, `the report of gitleaks is not valid`, or `gitleaks wrote an error message, so the scan is not complete`. A failed scan has no finding that an operator can confirm as a false positive. |
| `SECRET_SCAN_FAILED` or `COMMIT_MISMATCH` (after the commit) | `✘ The secret scan (gitleaks) of the change to store <store> did not complete: <reason>. Nothing was pushed.` or `✘ The commit of store <store> does not hold exactly the changed files: <reason>. Nothing was pushed.` Then `A scan refusal before staging writes no new Git object. A later failure can leave local objects. Do not assume that a reset removed them.` and ``Run `secrets store remove <name>`, then `secrets store clone <url>`.`` The `SECRET_SCAN_FAILED` form then has the two lines of a failed scan (`Next: secrets doctor` and `Then run the command again. …`). | The history or the tree of the new commit is not correct. The CLI found it before the push. Nothing was pushed, but local objects can stay in the clone. The repair is for the operator. |
| `COMMIT_MISMATCH` (before the scan) | `✘ The clone of store <store> is not at the base revision of this write. Nothing was committed or pushed.` | The clone changed during the write. The hint is `Run the command again.` |
| `UNSAFE_FILE` | `✘ Store <store> has a changed file that the CLI does not commit: <file label> <problem>. Nothing was committed or pushed.` | A changed file is not a readable regular file with a UTF-8 name and no symbolic link on its path, or it is larger than 256 MiB. When `lstat` cannot check a changed file (for example a name that is too long), the problem is `cannot be read`. The CLI never skips such a file as missing. |
| `UNSAFE_FILE` | `✘ Store <store> has a file that is not a regular file. Nothing was read.` (or `written`, `cloned`), `✘ Store <store> has a file that the CLI cannot check. Nothing was read.` (or `written`, `cloned`), or `✘ Store <store> has a file that is not a regular file, or that the CLI cannot check. …`. Then one line for each file: `- <file> is a symbolic link`, `- <file> is not a regular file`, or `- <file> cannot be checked (<code>)`. | A store file is a link or a special file, or `lstat` gave an error other than a missing path (`<code>` is a system error code, for example `EACCES`). The CLI reads and writes no such file, and it never treats a file that it cannot check as missing. |
| `UNSAFE_STORE` | `✘ A file of store <store> breaks the store policy. Nothing was written.` Then one line for each problem, for example `- dev.enc.yaml: line 26: the SOPS metadata field "version" does not have the form that SOPS writes`. Then two hints: `Each key must be encrypted (encrypted_regex ".*"), and the recipients of each file must be the recipients of its group in recipients.toml.` and ``This version has no migration command. Do not write to this store; create a new store with `secrets store init`.`` | Before the write, a store file breaks the store policy: a plaintext value, wrong recipients, or a SOPS metadata field (`version`, `lastmodified`, `mac`, `age.recipient`, `age.enc`, `encrypted_regex`) in a form that SOPS does not write. The line names the field, never its value. Nothing was written. |
| `UNSAFE_RESULT` | `✘ The result of a SOPS write breaks the store policy. Nothing was committed.` or `✘ The result of this write breaks the store policy. Nothing was committed.` Then one line for each problem, as for `UNSAFE_STORE`. Then ``The clone was reset to the remote. Check the SOPS version (`sops --version`).`` | After a SOPS write, the new file breaks the store policy (for example a metadata field in another form). The check runs before the scan and before the commit. Nothing was committed, and the clone was reset. |
| `COMMIT_FAILED`, `WRITE_FAILED` | `✘ The commit of store <store> stopped on a system error (<code>). Nothing was pushed.` or `✘ The write to store <store> stopped on a system error (<code>). Nothing was pushed.` | An unexpected error. The CLI shows at most a system error code such as `EACCES`, never a raw error or a path. `COMMIT_FAILED` also has the line `A scan refusal before staging writes no new Git object. …`. |
| (a warning) | `secrets: gitleaks is not on PATH, so the secret scan of the store commit did not run. Install gitleaks: pacman -S gitleaks (Arch Linux) or brew install gitleaks (macOS).` | The scan did not run, and the write continues. One warning for each command. |

The file label is a file name of the known store layout (`recipients.toml`, `.sops.yaml`, `.gitleaks.toml`, `README.md`, `.gitignore`, `dev.enc.yaml`, `preview.enc.yaml`, `prod.enc.yaml`, `deploy.enc.yaml`), or the fixed text `a file outside the known store layout`. A rule id is shown only in the form of a gitleaks rule id, else as a fixed text. The CLI never shows a match, a secret, a line, or the output of gitleaks.

Cleanup lines. The CLI checks each cleanup step. When a step fails, the error keeps its code and gets one more line:

- ``The CLI could not clean the clone of store <name>. Run `secrets store remove <name>`, then `secrets store clone <url>`.``
- `The CLI could not delete the new clone folder <folder>. Delete it.` (`store init`)
- `The CLI could not delete its private scan folder <folder>. Delete it.`

A scan refusal before staging writes no new Git object. A later failure can leave local objects. Do not assume that a reset removed them.

## Output rules

- Normal output never shows a value. The one exception is the stdout of `secrets export`. `secrets push` sends values to a platform and prints names only.
- Results go to stdout: tables, `added …` lines, the `doctor` report, the `plan` lines, the `push` lines, the public key of `keygen --public`.
- Diagnostics go to stderr: the `secret:` line, notices that start with `secrets:`, and errors. A CLI error starts with `✘`. It can have remedy lines, but not always. An error of the option parser starts with `error:`, for example `error: unknown option '--bad'` or `error: required option '--format <format>' not specified`.
- **Hints.** The lines under a `✘` line are hints. The output layer of the NUU CLIs (cli-core) prints a hint that starts with a command (``Run `<command>` …``) as `Next: <command>`. It prints a second command as `Or: <command>`. The rest of that line does not print. For example, ``Run `secrets doctor` to check gitleaks.`` prints as `Next: secrets doctor`. Thus the CLI puts each rule that must print in a hint of its own, for example `Then run the command again. Do not bypass a failed scan.` A hint that does not start with a command prints in full.
- Output names only public facts: variable names, secret names, store names, project names, recipient ids, public keys and key fingerprints, file names, paths, and line numbers.
- The CLI never shows the content of a store file, `.sops.yaml`, `.gitleaks.toml`, `recipients.toml`, `secrets.toml`, `secrets.local.toml`, `stores.toml`, a `.env` file, or the target of a symbolic link. SOPS and Git errors are a fixed summary. A URL is shown without its user information (`https://***@host/…`). A store URL that is not a usual form ([[dev-knw-nsec-setup]] § Store URL forms) shows as `<a store URL that the CLI cannot show>`, in each command.
- The CLI never shows the output of gitleaks. From the report, it shows only rule ids of the gitleaks form and its own file labels ([The secret scan of store commits](#the-secret-scan-of-store-commits)). In the write path, an unexpected error shows at most a system error code, never a raw error or a path.
- **Paths that the user chose.** Some error lines show a path that the user chose or set, for example `✘ No usable age key: no age key file at <path>.` and `✘ The mapping file <path> does not exist.` The output rules for values and file content do not cover the text of that path. An agent does not copy a line when its path or another field holds a value ([[dev-knw-nsec-rules]] § Rule 10, What to copy).

## Exit statuses

| Status | Meaning |
|---|---|
| `0` | Success. |
| `1` | An error. This includes the errors of the option parser: an unknown option, a missing required option, or an unknown command. `secrets doctor` exits 1 when a check fails. `secrets plan` exits 1 when the plan has an error. `secrets push` exits 1 for a plan error, a failed write, or an uncertain write. A store write exits 1 when the secret scan stops it (`SECRET_SCAN_FOUND`, `SECRET_SCAN_FAILED`). |
| `2` | With `✘ No command to run.`: `secret` or `secrets run` with no command. The program did not start. With `✘ secrets push <target> --env prod changes production. …`: `secrets push --env prod` without `--yes`. Nothing was read or sent. |
| `3` | A conflict in a write: another machine changed the same name to a different value. Nothing was written. (Writes are operator commands.) |
| `126` | `secret <cmd>`: the command is not executable. |
| `127` | `secret <cmd>`: the command is not on `PATH`. Any command: a required program (`sops`, `age-keygen`, `git`) is not on `PATH`. gitleaks is optional: when it is not on `PATH`, a store write prints a warning and continues. |
| `128 + n` | `secret <cmd>`: the signal `n` stopped the command. For example, 143 is SIGTERM. |
| other | `secret <cmd>` exits with the exit status of the command. |

After the command starts, `secret <cmd>` exits with the exit status of the command. That status can also be 1, 2, 3, 126, or 127. Thus read the message (`✘ …` or `error: …`), not only the status.

## Signals and tool children

- The CLI starts Git, SOPS, `age-keygen`, and gitleaks as child processes.
- Git, SOPS, and `age-keygen` stay in the process group of the CLI. Thus a prompt of the tool on the terminal still works: an SSH key passphrase, a host key question, or an age plugin PIN. The CLI tracks each of these children.
- On SIGINT, SIGTERM, or SIGHUP, the CLI gives each tracked child SIGKILL. Then it ends: exit status 130 for SIGINT, and the normal signal status for SIGTERM and SIGHUP. A descendant of a killed child (for example `ssh` under `git`) ends when its closed pipes stop it. The CLI does not wait for it.
- gitleaks runs in its own process group. The CLI kills the whole group at the time limit, at the end of the scan, and on SIGINT, SIGTERM, or SIGHUP.
- The editor of `secrets edit` and the command of `secret <cmd>` are not tracked. They keep the terminal and get the signal as before ([`secret <cmd>` and `secrets run`](#secret-cmd-and-secrets-run)).

## Messages that agents see

| Message (start) | Meaning | Action |
|---|---|---|
| `✘ <n> variable(s) have no value in env <env> of store <store>:` with lines `- <VAR> → secret://<name>` | The store has no value for these names in this environment. Nothing ran. | Ask the operator: [[dev-sk-nsec-request_value]]. |
| `✘ No usable age key: no age key file at <path>.` (or `the age key file <path> is not a regular file.`, or `the age key file <path> cannot be read.`) with the two lines `Next: secrets keygen` and `Or set SOPS_AGE_KEY_FILE to the path of an existing key file.` | The CLI found no usable key file before any decrypt. Nothing ran. `secrets keygen` makes a new key, which is another identity. Both lines are for the operator. | Tell the operator the path. Do not make a key. Use the key request of [[dev-sk-nsec-request_value]]. |
| `✘ sops decrypt failed (exit <n>) on <file>: the age key of this machine cannot decrypt the data key of the file.` | SOPS did not decrypt the file with the available key sources. This error does not prove that the selected key file is valid. The CLI does not show the cause. | Run `secrets doctor`. Do not run `sops`. |
| `✘ --offline: the store of <project> (<url>) has no cached clone on this machine. Nothing was cloned.` | `--offline` found no clone. Nothing ran. No network call. | When the task allows network access, run once without `--offline`, or `secrets store clone <url>`. Else report a blocker. |
| `secret: the store has no value for the optional <VAR>: the shell value is removed (--keep-env keeps it)` | The command runs without that variable. | Normal. Use `--keep-env` only when the task needs the shell value. |
| `✘ The stores folder <path> is a symbolic link. The CLI follows no symbolic link below the secrets home. Nothing was changed.` | `stores/` in the secrets home is a link. | Report the path to the operator. Do not move or link folders. |
| `✘ The CLI cannot check the stores folder <path> (<code>). Nothing was changed.` or `✘ The clone folder of store <name> is not safe to use: the CLI cannot check <path> (<code>). Nothing was changed.` (`UNSAFE_CLONE`) | `lstat` gave an error other than a missing path (`<code>` is a system error code, for example `EACCES`). The CLI does not treat the folder as missing. | Report the line to the operator. Do not change modes, and do not move, delete, or link folders in the secrets home. |
| `✘ There is no store on this machine and no secrets.toml here.` with `Next: secrets store clone <url>` and `Or: secrets store init <url> --recovery <age-public-key>` | No store clone and no mapping in this folder. | Go to the project folder, or use `--mapping` or `--project`. Else ask the operator which store to clone. Never run `store init` (an operator command). |
| `✘ The remote <url> is empty.` (`store clone`) or `✘ The remote of store <name> has no branch "<branch>".` (a refresh) | For `store clone`, the remote has no branch. For a refresh, the branch that the clone uses is missing. Other branches can still hold store commits. | Report it to the operator. The `store init` hint is for the operator. |
| `✘ Store <name> has a file that is not a regular file. Nothing was read.` (or `written`, `cloned`) with lines `- <file> is a symbolic link`, `- <file> is committed as a symbolic link (mode 120000)`, `- <file> is committed as a folder`, or `- <file> is not a regular file` | A store file is a link or a special file. The CLI reads and writes no store file through a link. | Report the lines to the operator. The repair is an operator task. |
| `✘ Store <name> has a file that the CLI cannot check. Nothing was read.` (or `written`, `cloned`; or `… is not a regular file, or that the CLI cannot check. …`) with lines `- <file> cannot be checked (<code>)` (`UNSAFE_FILE`) | `lstat` or the open of a store file gave an error other than a missing file. The CLI never treats such a file as missing. | Report the lines to the operator. Do not change modes or files in the store clone. |
| `secrets: warning: the dotenv npm package loses the variable __proto__ …` | `export --format dotenv` has a variable named `__proto__`. | Do not use the dotenv format for it. |
| `✘ <path> is not valid:` with `- targets.<target>: …` or `- env.<VAR>.to: "convex" is not allowed: the Convex tool owns <VAR>, …` | A `[targets.*]` table or a Convex target is not valid. Every command of the project stops, also `secret <cmd>`, `run`, and `export`. | Run `secrets doctor`. Fix the table only when the operator or the task gives its content (Rule 8). Else report it. |
| `✘ <path> has no [targets.<target>] table.` | `plan` or `push` for a target with no table. | Report it. Do not add the table on your own (Rule 8). |
| `✘ --env deploy: plan and push deliver dev, preview, prod.` | `deploy.enc.yaml` holds the credentials and is never delivered. | Use `dev`, `preview`, or `prod`. |
| `error    this machine cannot open deploy.enc.yaml: …` (a `plan` line) | The CLI cannot read `deploy.enc.yaml`. The reason can be absent deploy membership or a decrypt error. The credentials show `no access`. | Report the fixed reason. Do not infer membership from this status alone. It is not permission to add access. |
| `✘ secrets push <target> --env prod changes production. Each push to production needs the approval of the operator. Nothing was read or sent.` with `Next: secrets plan <target> --env prod` and `With the approval of the operator, run the push again with --yes.` | `push --env prod` with no `--yes` (exit 2). | Agents do not run `push` and never add `--yes`. Give the operator the `plan` and `push` commands ([[dev-sk-nsec-plan_push]]). |
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
| `✘ The secret scan (gitleaks) found <n> possible secret(s) in the change to store <store>.` | `SECRET_SCAN_FOUND`: a store write stopped (exit 1). | Stop and report. Follow [[dev-knw-nsec-rules]] § Rule 10. Never use the `SECRETS_GITLEAKS=0` hint. |
| `✘ The secret scan (gitleaks) of the change to store <store> did not complete: <reason>.` with `Next: secrets doctor` and `Then run the command again. Do not bypass a failed scan.` | `SECRET_SCAN_FAILED`: a store write stopped (exit 1). | Run `secrets doctor` and report the fixed reason. Follow [[dev-knw-nsec-rules]] § Rule 10. |
| `✘ A file of store <store> breaks the store policy. Nothing was written.` (`UNSAFE_STORE`) or `✘ The result of a SOPS write breaks the store policy. Nothing was committed.` (`UNSAFE_RESULT`), with a line such as `- dev.enc.yaml: line <n>: the SOPS metadata field "<field>" does not have the form that SOPS writes` | A store file, or the result of a SOPS write, breaks the store policy. The line names the field only, never its value. | Stop and report the file label, the line number, and the field name to the operator. Do not retry the write. Do not open the store file. Follow [[dev-knw-nsec-rules]] § Rule 10. |
| `secrets: gitleaks is not on PATH, so the secret scan of the store commit did not run. …` | The store write had no scan. | Check the exit status, and report both facts. Follow [[dev-knw-nsec-rules]] § Rule 10. |
| `✘ secrets setup needs a terminal on stdin and stdout. Nothing was changed.` with `Next: secrets setup --print` | `secrets setup` with no terminal (exit 1, `NO_TERMINAL`). | Do not run `secrets setup`. Run `secrets setup --print`, and give the operator the operator commands ([[dev-knw-nsec-setup]]). |
| `<a store URL that the CLI cannot show>` | A store URL that is not a usual form. The CLI never shows it. | Ask the operator for the URL in a usual form. To fix a mapping: `secrets project set --store <url>`. To fix a clone: the operator runs `secrets store remove <name>`, then `secrets store clone <url>`. |
| `✘ --store: a URL that the CLI cannot check (…)` or `✘ --store: an HTTP URL with user information (a token). …` (`BAD_STORE_URL`) | `init-project` or `project set` refused the URL. Nothing was written. | Ask the operator for a URL in a usual form with no credential. Git gets the credential from its helper or the SSH key. |
| `✘ secrets <cmd> cannot edit this secrets.toml: <reason>. Nothing was written.` (`MAPPING_EDIT`) | A setup command cannot edit an unusual form of the file (for example dotted keys or a multi-line value). | Report it to the operator, with the reason. |
| `✘ Git does not ignore <path>. Nothing was written.` with `Next: secrets project gitignore`, or `✘ Git tracks <path>. Nothing was written.` (`NOT_IGNORED`) | `bind --local` refused: Git does not ignore `secrets.local.toml`, or tracks it. | Run `secrets project gitignore`. For a tracked file, run `git rm --cached secrets.local.toml` and tell the operator: the Git history keeps the old file. |
| `✘ <n> [env] entry has <t> in to: <VAR>, …. Nothing was written.` (`TARGET_IN_USE`) | `target rm` while entries still use the target. | Remove the target from each entry with `bind` (the same name, the other targets) only when the operator or the task says so. |
| `✘ <VAR> is not in [env] of <path>. Nothing was written.` (`NO_ENTRY`) | `unbind` of a variable with no entry. | Check the name with `secrets vars`. |
| `✘ <path> is a symbolic link now, …` or `✘ <path> goes through the symbolic link <link> now, and projects.toml has the identity <name> for that path. …` (`LINKED_MAPPING`) | The path of the `secrets.toml` has a link, and the project index has an identity for that path. The CLI uses neither that identity nor the default. Nothing was read. | Stop. Give the operator the message and its remedy lines ([[dev-knw-nsec-rules]] § Rule 12). |
| `✘ The CLI cannot prove that <path> is the secrets.toml that it read: the file changed during the command. …` (`MAPPING_CHANGED`) | The file or its path changed during the command, and its entry has an identity. Nothing was read. | Stop. Give the operator the message and its remedy lines. |
| `✘ <path> is outside the project of the current folder (<boundary>). The identity <name> of that secrets.toml in projects.toml applies only in its own project …` (`OUTSIDE_PROJECT`) | The search found the `secrets.toml` of another project above the Git repository of the current folder. Nothing was read. | Stop. Give the operator the message and its remedy lines. Do not change the folder or `--mapping` to pass the check. |
| `✘ projects.toml has <n> entries for one secrets.toml (<path>), with different identities:` with one line for each entry (`DUPLICATE_PROJECT`) | An index of an earlier version has two entries of one file. Nothing was read. | Stop. Give the operator the message and its remedy lines (`secrets project identity <name> --mapping <path>`, `secrets project forget <path>`, or `--identity <name>`). |
| `✘ There is no identity named "<name>" (<where>).` (`UNKNOWN_IDENTITY`) with `Next: secrets identity list` | `--identity`, `SECRETS_IDENTITY`, the project identity, or the default names an identity that `identities.toml` does not have. `<where>` names the source. | Stop. Run `secrets identity list`, and give the operator both outputs. |
| `✘ The CLI cannot read the identity of project <name>: …` (`PROJECT_INDEX`) | `projects.toml` is not valid or not a regular file. | Stop. Run `secrets project list`, and give the operator both outputs. Do not edit `projects.toml`. |
| `secrets: warning: <path> has no identity of its own in projects.toml, so the CLI uses <identity or key file>. The secrets.toml above it in the same project, <path> (project <name>), has the identity <name>. …` | The nearest `secrets.toml` has no project identity, but a `secrets.toml` above it has one. The command did not stop: it used the default. | After the command, stop and give the operator the warning ([[dev-knw-nsec-rules]] § Rule 12). |
| `secrets: warning: …, so the identity <name> of its target <path> in projects.toml does not apply (a link can name the secrets.toml of another project). …` | The path of the `secrets.toml` has a link. The command did not use the identity of the link target. | After the command, stop and give the operator the warning. |
| `secrets: warning: <path> has an identity key, and the CLI ignores it: a project file never selects an identity. …` | `secrets.local.toml` has an `identity` key. | Report it. The operator sets the project identity and removes the line. |
| `✘ secrets ui needs a terminal on stdin and stdout: the UI is for a person at a terminal. Nothing was started.` (`NO_TERMINAL`) | An agent ran `secrets ui`. | Never run it again, and never give it a terminal. Use `secret <cmd>`, and ask the operator for the values ([[dev-sk-nsec-request_value]]). |

## Known limits

These limits are in the current build (the CLI of Mission 002 on `even-macbook`, not yet on canon; items 1 to 22 also apply to canon `471e270` and npm `0.1.0-beta.1`).

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
12. **The secret scan covers only the commits of the CLI.** A manual commit in another checkout of the store has no scan. Change the managed store only through `secrets`.
13. **The `ok` line of gitleaks is not a capability proof.** `doctor` proves only that `gitleaks version` starts. The scan is tested with gitleaks 8.30.1. An older gitleaks can stop each write that needs a commit (`SECRET_SCAN_FAILED`).
14. **A false positive is possible.** The tests with gitleaks 8.30.1 found no false positive in SOPS ciphertext, age stanzas, or the MAC. A different value or rule can still produce a false positive. Only the operator decides that a finding is a false positive.
15. **A late failure can leave local objects.** A scan refusal before staging writes no new Git object. A later failure can leave local objects in the store clone. Do not assume that a reset removed them.
16. **A hint shows only its command.** cli-core prints a hint that starts with a command as `Next: <command>` or `Or: <command>`, and the rest of that line does not print ([Output rules](#output-rules), Hints). The CLI puts each rule that must print in a line of its own. For example, a failed scan prints `Next: secrets doctor` and `Then run the command again. Do not bypass a failed scan.` The rule is the same: do not bypass a failed scan.
17. **`secrets vars` finds only its read forms.** A name that the code makes at run time is not found. A name in a comment counts. Names that a framework sets itself can show as `unmapped`.
18. **`secrets setup` exits with the status of `doctor`.** A complete setup exits 1 while a check fails, for example while a required prod value is missing.
19. **The recovery key path check cannot find each work tree.** The setup refuses a path below a `.git` entry and asks Git. A work tree that Git finds only through `GIT_DIR` (for example a bare repository of dotfiles with the home folder as its work tree) has no `.git` entry, and the check cannot find it.
20. **A deleted recovery key file can stay on the disk.** On an SSD or a copy-on-write file system, a deleted file can stay readable. The paper is the recovery key.
21. **The store URL forms are strict.** The CLI refuses, and does not show, each store URL that is not a usual form ([[dev-knw-nsec-setup]] § Store URL forms), also some forms that Git accepts, for example `ssh://git@my_host/x` (a `_` in a URL host) or a path with `?` or `#`. A hint for an SSH URL with a user name shows `***@`, so that hint command is not usable as it is.
22. **`secrets setup --print | head` ends with an error.** When the reader of stdout closes early, the command ends with an `EPIPE` error and exit 1. This is also true for `doctor`.
23. **An identity is not a security boundary.** Each program of one operating-system user can read each key file of that user and can change `identities.toml` and `projects.toml`. For a boundary, the operator uses another operating-system user or another machine.
24. **`SOPS_AGE_KEY_FILE` has no isolation.** Step 3 of the key selection keeps the old behavior: SOPS can also use its other key sources. Only a named identity (steps 1, 2, 4, and 5) gets its key file alone.
25. **A folder link in a cloned repository.** When the shell is in a folder link, the operating system gives the CLI the physical folder, so the CLI uses the `secrets.toml` and the identity of the target. But a child process of `secret <cmd>` gets the logical `PWD` of the shell. Run `secret <cmd>` from the real folder of the project.
26. **A key file with a passphrase is not supported.** `secrets identity add --key` refuses it.
27. **Non-Git projects.** The project boundary of a folder with no Git repository is the current folder. In a subfolder of such a project, the `secrets.toml` above is outside that boundary: when its entry has an identity, the command stops (`OUTSIDE_PROJECT`). The operator runs the command in the folder of the `secrets.toml`, or uses `--mapping`.
