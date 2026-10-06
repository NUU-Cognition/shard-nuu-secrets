---
description: "The store model and the project mapping: store files and groups, secret names, secret:// references, secrets.toml and secrets.local.toml entries, targets, literals, the environment chain, and what goes in the store"
---

# Knowledge: Store Model and Project Mapping

This file describes where the values are, how a project names them, and which values go into a process. The words are the words of [[dev-knw-nsec-rules]] § Words.

## The store

One team has one store: a private Git repository. Each value in it is encrypted. The secret names, the recipients, and the SOPS policy metadata are plaintext.

| File | Content | Group |
|---|---|---|
| `recipients.toml` | The recipients (age public keys) and their groups. Plaintext. | — |
| `.sops.yaml` | The SOPS policy. The CLI makes it from `recipients.toml`. | — |
| `.gitleaks.toml` | The gitleaks configuration. The CLI makes it. It keeps the default rules of gitleaks and allows only the recipient key lines of `recipients.toml` (`key = "age1…"`). A token in another file, and any other token in `recipients.toml`, is still a finding. The secret scan of the CLI uses the same generated text, but it does not read this file. An existing store needs one `secrets store fix-config` by the operator to get the text of canon `35ecaba` ([[dev-knw-nsec-cli]] § The config of stores). | — |
| `README.md`, `.gitignore` | Plain files of the store. | — |
| `dev.enc.yaml` | The development values. | `dev` |
| `preview.enc.yaml` | The preview values that are different from dev. | `dev` |
| `prod.enc.yaml` | The production values. | `prod` |
| `deploy.enc.yaml` | The platform credentials of `secrets plan` and `secrets push`: `vercel/token`, `vercel/team-id`, `convex/<deployment>/<env>`, `railway/token`. | `deploy` |

- SOPS encrypts each value with the data key of its file. The data key is encrypted for each recipient of the group of the file.
- A recipient is the age key of one machine, with an id `<Name>@<machine>` and a list of groups. A machine opens only the files of its groups.
- The recovery key is an age key on paper. It is a recipient of all groups.
- The CLI keeps the store clones in the secrets home: `$SECRETS_HOME`, else `<NUU home>/secrets/` for the prod build and `<NUU home>/secrets-dev/` for the dev build. The NUU home is `$NUU_HOME`, else `~/.nuucognition`. The list of clones is `<secrets home>/stores.toml`.
- The secrets home and its ancestors can be symbolic links. Commands that check a managed clone refuse linked `stores/`, clone, or `.git` folders below the real secrets home. Not every metadata command runs that check. A successful `whoami` is not proof that these folders passed it. Use plain `secrets doctor` to check the clone.
- Each file of a store that the CLI reads or writes must be a regular file: `recipients.toml`, `.sops.yaml`, `.gitleaks.toml`, `README.md`, `.gitignore`, and each `*.enc.yaml`. The CLI checks the commit (mode `120000` is a link) and the work tree. It never reads or writes through a link, and it refuses a FIFO or another special file at once. A file or a folder that the CLI cannot check (an `lstat` error other than a missing path) is never treated as missing: the command stops.
- The SOPS metadata of each store file (`version`, `lastmodified`, `mac`, the `age` entries, and `encrypted_regex`) must have the form that SOPS writes. A store write stops on a field in another form (`UNSAFE_STORE` before the write, `UNSAFE_RESULT` after a SOPS write). The message names the field, never its value.
- The store clone is a cache that the CLI owns. A refresh (`secret`, `export`, `plan` without `--offline`, `push`, `store sync`, `doctor --online`) fetches it and resets it to the remote.
- A write that changes the store makes one Git commit and one push. A write with no change makes no commit. The operator does the writes, also `store fix-config` and `store remove` (see [[dev-knw-nsec-rules]] § Rule 5).
- Before a store commit, the CLI scans the exact bytes of the change unless gitleaks is missing or `SECRETS_GITLEAKS=0`. No Git hook runs in the store clone. The scan does not cover a manual commit in another checkout, so the team changes the store only through `secrets`. See [[dev-knw-nsec-cli]] § The secret scan of store commits and [[dev-knw-nsec-rules]] § Rule 10.

You never open these files. The CLI reads them for you. See [[dev-knw-nsec-rules]] § Rule 4.

## Secret names

A secret name is a lowercase path, for example `clerk/secret-key`. The path is the YAML path in the store file.

- Each segment matches `[a-z0-9][a-z0-9_-]*`. Segments are separated by `/`.
- A name has 200 characters or fewer.
- A segment cannot end with `_unencrypted`. The first segment cannot be `sops`. SOPS gives these keys a special meaning.
- The environment is not part of the name. The environment selects the store file. Thus `clerk/secret-key` in `dev.enc.yaml` and `clerk/secret-key` in `prod.enc.yaml` are two values with one name.
- In `secrets.toml`, a reference to a name is `secret://<name>`, for example `secret://clerk/secret-key`.

## The environment chain

`--env` selects the environment. The default is `dev`. The CLI reads these files, in this order. A later file replaces the names of an earlier file.

| `--env` | Files that the CLI reads | Group that this machine needs |
|---|---|---|
| `dev` | `dev.enc.yaml` | `dev` |
| `preview` | `dev.enc.yaml`, then `preview.enc.yaml` over it | `dev` |
| `prod` | `prod.enc.yaml` only | `prod` |
| `deploy` | `deploy.enc.yaml` only | `deploy` |

- `preview` is a sparse overlay. A name that is only in `dev.enc.yaml` gets its dev value in `preview`.
- `prod` never falls back to `dev`. A name that is only in `dev.enc.yaml` has no value in `prod`.
- `deploy` holds platform credentials. It does not fall back.

## The project mapping: `secrets.toml`

Each project has one committed `secrets.toml`. It holds names and targets, never values.

```toml
[project]
name = "nuu-track"
store = "git@github.com:NUU-Cognition/nuu-secrets.git"   # a Git URL or a local path

[targets.vercel]
project = "nuu-track"

[env]
CLERK_SECRET_KEY = { ref = "secret://clerk/secret-key", to = ["local", "vercel"] }
SENTRY_DSN       = { ref = "secret://sentry/dsn", to = ["local"], optional = true }
LOG_LEVEL        = { value = "info", to = ["local"] }
```

| Part | Rule |
|---|---|
| `[project]` | Two fields only: `name` (a letter or a digit first, then letters, digits, `.`, `_`, or `-`; 100 characters or fewer) and `store` (the Git URL or the path of the store). The URL must not hold a credential: `secrets init-project` refuses such a URL. |
| `[targets.<target>]` | One table for each platform target: `vercel`, `convex`, or `railway`. See [The target tables](#the-target-tables). |
| `[env]` | One entry for each variable. The key is the variable name: letters, digits, and `_`, and not a digit first. |
| `ref` | A reference `secret://<name>` to a value in the store. |
| `value` | A literal: a value that is not secret. It is committed in plaintext. |
| `to` | A list of targets, without repeats: `local`, `vercel`, `convex`, `railway`. |
| `optional` | `true` or `false`. Only for a `ref` entry. |

- Each entry has exactly one of `ref` or `value`. The fields are `ref`, `value`, `to`, and `optional`. Another field is an error.
- `local` is the target of `secret <cmd>` and of `secrets export` (the default `--to local`). A platform target needs its `[targets.<target>]` table.
- When the environment has no value for an `optional` entry, the CLI does not give that variable and does not stop. The `secret:` line counts it as "optional not set". The mapping owns the name: `run` also removes a shell variable of the same name, and stderr names it. With `--keep-env`, the shell value stays.
- A `ref` entry with no value, and not `optional`, is an error. The error names the variable and the secret name. Nothing runs.
- The CLI finds `secrets.toml` in the current folder or in a parent folder. `--mapping <path>` gives another file. `--project <name>` finds the `secrets.toml` with that project name under the root of the current Git repository. It skips `node_modules`, `.git`, and `dist*`.
- The `store` of the mapping selects the store. When this machine has no clone of that store, `secret` clones it (a network access) and writes `secrets: cloning the store of <project> from <url>` on stderr. With `--offline`, it never clones: it stops with exit 1.

## The target tables

`secrets plan` and `secrets push` read the `[targets.<target>]` table of each platform target. The CLI checks each table when it loads the mapping.

```toml
[targets.vercel]
project = "nuu-track"                    # a Vercel project name or id

[targets.convex]
deployment = "track"                     # one name segment: the deploy key is convex/track/<env>

[targets.railway]
project      = "nuu-discord-bot"         # a Railway project name or id
service      = "bot"                     # a Railway service name or id
environments = { prod = "production" }   # store environment → Railway environment name or id
```

- `[targets.vercel]` has `project` only. The team id comes from `vercel/team-id` in `deploy.enc.yaml`: a `team` field is an error.
- `[targets.convex]` has `deployment`: one name segment (lowercase letters, digits, `_`, and `-`).
- `[targets.railway]` has `project`, `service`, and `environments` (at least one entry; the keys are `dev`, `preview`, and `prod`). An environment with no entry cannot go to Railway.
- A missing field, an unknown field, or a bad name is an error of the mapping. A table that is not valid stops **every** command of that project, also `secret <cmd>`, `run`, and `export`. Run `secrets doctor`: it names each problem and never shows a value.
- The Convex tool owns some names, and the CLI never writes them to a Convex deployment: `CONVEX_DEPLOYMENT`, `CONVEX_DEPLOY_KEY`, `CONVEX_DEPLOYMENT_TOKEN`, `CONVEX_SELF_HOSTED_URL`, `CONVEX_SELF_HOSTED_ADMIN_KEY`, `CONVEX_CLOUD_URL`, `CONVEX_URL`, `CONVEX_SITE_URL`, and the client URL names (`NEXT_PUBLIC_CONVEX_URL`, `VITE_CONVEX_URL`, `PUBLIC_CONVEX_URL`, `REACT_APP_CONVEX_URL`, `EXPO_PUBLIC_CONVEX_URL`, and the same `…_CONVEX_SITE_URL` names). An entry with one of these names cannot have the target `convex`. Another target (for example `NEXT_PUBLIC_CONVEX_URL` to `vercel`) is valid.
- A platform target and its table are operator decisions ([[dev-knw-nsec-rules]] § Rule 8). The delivery is in [[dev-sk-nsec-plan_push]].

## The local mapping: `secrets.local.toml`

`secrets.local.toml` is an optional file beside `secrets.toml`. It is for this machine only.

- It must be in `.gitignore`. `secrets doctor` fails when Git tracks it.
- It can hold only an `[env]` table. Each entry can target only `local`.
- For the `local` target, its entries replace the entries of `secrets.toml` with the same variable.
- `secrets export --to <platform target>` does not use its entries in the output. But the CLI still reads and checks the file: a `secrets.local.toml` that is not valid stops that export too (exit 1).
- `secrets bind` writes only `secrets.toml`. When `secrets.local.toml` has the same variable, `bind` writes a notice on stderr: the local entry wins on this machine.

## Literals

A literal (`value = "…"`) is committed in plaintext. Use a literal only when both of these are true:

1. The value is not secret.
2. The value is the same in all environments.

A literal has no environment. `--env prod` gives the same literal as `--env dev`. A value that is different in each environment goes in the store, also when it is not secret.

A literal of `secrets.toml` is public. `secrets push` does not check literals against the ids of a platform, but it checks each store value of any length ([[dev-knw-nsec-cli]] § Known limits, item 10). Thus a short value that is not secret and is the same in all environments is better as a literal. The operator decides. Never move a secret value into `secrets.toml`.

`secrets bind` refuses a literal that looks like a secret, and `secrets doctor` gives `fail` for it. The check looks for a known key prefix (for example `sk_live_`), a key block, a JWT, a URL with a password, a high-entropy string of 24 or more characters, and a random token of 16 to 23 letters and digits. A UUID and a 40-character lowercase hex string pass, except when a word of the variable name says secret (for example `TOKEN` in `POSTMARK_API_TOKEN`). The check does not find all secrets ([[dev-knw-nsec-cli]] § Known limits, item 2). You decide first: when a value can be secret, it is not a literal.

## What goes in the store, and what does not

| Kind of variable | Where it goes |
|---|---|
| A secret: an API key, a token, a password, a signing key, a URL with a credential | The store. A `ref` entry. |
| A value that is not secret, but is different in each environment | The store. A `ref` entry. |
| A value that is not secret and is the same in all environments | A literal in `secrets.toml`. |
| A different entry for this machine only | An entry in `secrets.local.toml`: a `ref` to another secret name, or a literal that is not secret. Never a value. |
| A variable that another tool writes and manages, for example `CONVEX_DEPLOYMENT` (written by `npx convex dev` into `.env.local`) | Not in the store and not in the mapping. The tool owns it. |

- A `NEXT_PUBLIC_*` variable is for the browser build: it is public in the bundle. `secrets doctor` warns when a `NEXT_PUBLIC_*` entry targets `convex`.
- A variable of a `.env*` file that the mapping also manages is a duplicate. `secrets doctor` warns and names it. The operator removes it from the `.env*` file: `secret <cmd>` gives the variable.

## Process environment of `secret <cmd>`

- The process gets the shell environment and the variables of the `local` target.
- A store value replaces a shell variable of the same name. stderr names each replaced variable (never the value). `--keep-env` keeps the shell value.
- An `optional` variable with no store value: `run` removes a shell variable of the same name. `--keep-env` keeps it.
- The values stay in memory and in the process environment of the command. `secret` writes no file with a plaintext value. A refresh or a first clone changes the Git files and the index of the store clone in the secrets home (ciphertext and public data only).
- Other processes of the same user, and root, can read the process environment of the command (`/proc/<pid>/environ`).
