---
description: "The store model and the project mapping: store files and groups, identities and the key selection order, the project index, secret names by store project, the shared project, links and implicit refs, declared environments, tags, expiry dates, the change log and moved refs, the vault, secrets.toml and secrets.local.toml entries, targets, literals, the environments (each reads only its own store file), the block of .env.local that secrets pull writes, and what goes in the store"
---

# Knowledge: Store Model and Project Mapping

This file describes where the values are, how a project names them, and which values `secrets pull` writes into the block of `.env.local`. The words are the words of [[dev-knw-nsec-rules]] § Words.

## The store

One team has one store: a private Git repository. Each value in it is encrypted. The secret names, the recipients, and the SOPS policy metadata are plaintext.

| File | Content | Group |
|---|---|---|
| `store.toml` | The store format: one line, `format = 2`. The CLI reads it before each read of values and each write, and refuses a store of another format (`STORE_FORMAT`). A store with no `store.toml` has the store format 1 (the previous version). Plaintext. | — |
| `recipients.toml` | The recipients (age public keys) and their groups. Plaintext. | — |
| `.sops.yaml` | The SOPS policy. The CLI makes it from `recipients.toml`. | — |
| `.gitleaks.toml` | The gitleaks configuration. The CLI makes it. It keeps the default rules of gitleaks and allows only the recipient key lines of `recipients.toml` (`key = "age1…"`). A token in another file, and any other token in `recipients.toml`, is still a finding. The secret scan of the CLI uses the same generated text, but it does not read this file. An existing store needs one `secrets store fix-config` by the operator to get the text of canon `35ecaba` ([[dev-knw-nsec-cli]] § The config of stores). | — |
| `README.md`, `.gitignore` | Plain files of the store. | — |
| `dev.enc.yaml` | The development values. | `dev` |
| `preview.enc.yaml` | The preview values. `preview` reads only this file. | `dev` |
| `prod.enc.yaml` | The production values. | `prod` |
| `deploy.enc.yaml` | The platform credentials of `secrets plan` and `secrets push`: `shared/VERCEL_TOKEN`, `shared/VERCEL_TEAM_ID`, `shared/RAILWAY_TOKEN`, and `<project>/CONVEX_DEPLOY_KEY_DEV`, `…_PREVIEW`, `…_PROD`. | `deploy` |
| `tags.toml` | The tag records: a description, a source URL, and a warning window for each tag that has a record. Plaintext, no values. | — |
| `metadata.toml` | The declared environments of each secret whose set is not the default, the tags of the secrets and of the vault entries, and the expiry dates of the secrets, for each environment. Names and tags only, no values. | — |
| `changes.jsonl` | The change log: one JSON line for each change of a store write. Names only, never a value or a vault text. | — |
| `vault/<group>/<name>.enc.yaml` | One vault entry: a free text, encrypted. The name is the path of the file. | `dev` or `prod` |

- SOPS encrypts each value with the data key of its file. The data key is encrypted for each recipient of the group of the file.
- A recipient is the age key of one machine, with an id `<Name>@<machine>` and a list of groups. A machine opens only the files of its groups.
- The recovery key is an age key on paper. It is a recipient of all groups.
- The CLI keeps the store clones in the secrets home: `$SECRETS_HOME`, else `<NUU home>/secrets/` for the prod build and `<NUU home>/secrets-dev/` for the dev build. The NUU home is `$NUU_HOME`, else `~/.nuucognition`. The list of clones is `<secrets home>/stores.toml`. The secrets home also holds `identities.toml` (the identities), `projects.toml` (the project index), `keys/` (the new keys of `secrets identity add`), and `ui-reveal.log` (the reveal log of `secrets ui`).
- The secrets home and its ancestors can be symbolic links. Commands that check a managed clone refuse linked `stores/`, clone, or `.git` folders below the real secrets home. Not every metadata command runs that check. A successful `whoami` is not proof that these folders passed it. Use plain `secrets doctor` to check the clone.
- Each file of a store that the CLI reads or writes must be a regular file: `recipients.toml`, `.sops.yaml`, `.gitleaks.toml`, `README.md`, `.gitignore`, `tags.toml`, `metadata.toml`, `changes.jsonl`, each `*.enc.yaml`, and each vault file. Git must commit `tags.toml`, `metadata.toml`, `changes.jsonl`, and each vault file: an ignore rule of Git that matches one of them stops each write (`UNSAFE_STORE`). The CLI checks the commit (mode `120000` is a link) and the work tree. It never reads or writes through a link, and it refuses a FIFO or another special file at once. A file or a folder that the CLI cannot check (an `lstat` error other than a missing path) is never treated as missing: the command stops.
- Before each decrypt, the CLI reads the store file itself and checks its SOPS metadata. The same check runs before each decrypt of a vault file. A key of another system (`kms`, `gcp_kms`, `hckms`, `azure_kv`, `hc_vault`, `pgp`, or `key_groups` that is not empty), an unknown metadata key, a YAML problem, or no SOPS metadata stops the read before SOPS runs (`UNSAFE_STORE` for a store file, `UNSAFE_VAULT` for a vault file). Only age keys open a store file ([[dev-knw-nsec-cli]] § The check of the SOPS metadata before each decrypt).
- The SOPS metadata of each store file (`version`, `lastmodified`, `mac`, the `age` entries, and `encrypted_regex`) must have the form that SOPS writes. A store write stops on a field in another form (`UNSAFE_STORE` before the write, `UNSAFE_RESULT` after a SOPS write). The message names the field, never its value.
- The store clone is a cache that the CLI owns. A refresh (`pull`, `export`, `plan` without `--offline`, `push`, `store sync`, `doctor --online`) fetches it and resets it to the remote.
- A write that changes the store makes one Git commit and one push. A write with no change makes no commit. The operator does the writes, also `store fix-config` and `store remove` (see [[dev-knw-nsec-rules]] § Rule 5).
- Before a store commit, the CLI scans the exact bytes of the change unless gitleaks is missing or `SECRETS_GITLEAKS=0`. No Git hook runs in the store clone. The scan does not cover a manual commit in another checkout, so the team changes the store only through `secrets`. See [[dev-knw-nsec-cli]] § The secret scan of store commits and [[dev-knw-nsec-rules]] § Rule 10.

You never open these files. The CLI reads them for you. See [[dev-knw-nsec-rules]] § Rule 4.

## Identities and the key selection

An identity is a name for one age key file of this machine. For example, `work` is the key that opens the store of the team, and `client` is the key of the store of a client. The identities are in `identities.toml` in the secrets home. The file holds names and key file paths, never a key. A new key of `secrets identity add <name>` goes to `<secrets home>/keys/<name>.txt` (mode 600).

**The selection order.** For each command that decrypts or reads the key, the CLI selects one key file. The first match wins:

1. `--identity <name>` (on each command that decrypts, and on `doctor`, `setup`, `keygen`, `whoami`, and `store init|clone`).
2. The variable `SECRETS_IDENTITY`.
3. The variable `SOPS_AGE_KEY_FILE` (a key file, not an identity).
4. The identity of the project in `projects.toml` of this machine (set with `secrets project identity <name>`).
5. `default` in `identities.toml`.
6. The default key file `~/.config/sops/age/keys.txt`.

- Steps 4 and 5 need `identities.toml`. With no `identities.toml`, the CLI selects `SOPS_AGE_KEY_FILE`, else the default key file, as before.
- `secrets doctor` shows the step that selected the key: `key selection: <reason>`. `secrets whoami` shows `identity <name> (<reason>)`. `secrets identity list` ends with `active: <reason>  key file <path>`.
- **A named identity gets only its key file.** With an identity (steps 1, 2, 4, and 5), each SOPS process gets the key file of the identity and no other key source: no `SOPS_*` variable of the shell and no default age or SSH key. So an identity opens only the store files of its groups. A missing key file of an identity is an error, also when SOPS has another key source.
- **A project file never selects an identity.** The identity of a project is in `projects.toml` of the machine, not in `secrets.toml` or `secrets.local.toml`. The CLI ignores an `identity` key in `secrets.local.toml` and prints one warning. So a cloned project cannot select an identity.
- **Where step 4 applies.** The project identity applies only to the `secrets.toml` of the project that you are in: a path with no symbolic link, inside the project boundary of the current folder (the top level of its Git repository, else the current folder), and the same file at the read and at the lookup. Else the command stops when the entry has an identity (`LINKED_MAPPING`, `OUTSIDE_PROJECT`, `MAPPING_CHANGED`), because the default can have more access. Two old entries of one file with different identities also stop the command (`DUPLICATE_PROJECT`).
- **The parent warning.** When step 4 gives no identity for the nearest `secrets.toml`, and a `secrets.toml` above it in the same project has an identity, the CLI prints one warning and does not stop. The command then uses the default identity (or the default key file).
- **The least access is the default.** The README of the CLI advises the operator to make the identity with the least access the default. Then a project identity gives more access only in the projects where the operator set it.
- **Not a security boundary.** Each process of one operating-system user can read each key file of that user and can change `identities.toml` and `projects.toml`.

The agent rules for identities are in [[dev-knw-nsec-rules]] § Rule 12. The setup for the operator is in [[dev-knw-nsec-setup]] § Identities.

## The project index

`projects.toml` in the secrets home lists the projects of this machine. Each command that loads a `secrets.toml` adds or updates its entry (`secrets setup --print` does not). A command that decrypts records the project only after its key check. `secrets project add <folder>` adds the projects under a folder that no command loaded yet: a new entry has no identity, and a known entry keeps its identity ([[dev-knw-nsec-cli]] § `secrets project add`).

- An entry has the path of the `secrets.toml`, the project name, the store name, the project identity (step 4), and the last time that a command loaded the file. It holds no value.
- `secrets project list` shows each entry with a state: `ok`, `missing` (the path does not exist), `linked` (the path has a symbolic link: its identity does not apply), or `duplicate` (another entry is for the same file, for example another letter case or Unicode form of the path on macOS).
- `secrets project forget <name|path>` removes the entries of a project. It changes no project file. It is an operator command ([[dev-knw-nsec-rules]] § Rule 5): an entry can hold the project identity.

## Secret names

A secret name is `<project>/<VARIABLE>`, for example `nuu-track/CLERK_SECRET_KEY` or `shared/CLERK_SECRET_KEY`. The name is the YAML path in the store file (`nuu-track:` holds `CLERK_SECRET_KEY:`).

- A name has exactly two segments.
- The project segment is a store project: the `[project] name` of one `secrets.toml`, or the special project `shared`. It follows the rule of a project name: letters, digits, `.`, `_`, and `-`, a letter or a digit first, 100 characters or fewer.
- The variable segment follows the rule of an environment variable name: letters, digits, and `_`, not a digit first. Inside its project, the name of a secret is the name of its variable.
- A segment cannot end with `_unencrypted`, and the project cannot be `sops`. SOPS gives these keys a special meaning.
- The environment is not part of the name. The environment selects the store file. Thus `nuu-track/CLERK_SECRET_KEY` in `dev.enc.yaml` and in `prod.enc.yaml` are two values with one name.
- In `secrets.toml`, a reference to a name is `secret://<name>`, for example `secret://shared/SENTRY_DSN`.
- `secrets list` shows the names by project, with the environments that have a value, the tags, and the projects of this machine that use each one. `--project <name>` shows one project and its links to `shared`; `--project shared` shows each shared secret and the projects that link to it. It reads plaintext names and needs no key.

## Store projects and `shared`

Each store project matches one local `secrets.toml`: the project of `nuu-track/CLERK_SECRET_KEY` is the project with `[project] name = "nuu-track"`.

- **The shared project.** A secret in `shared` is the only copy of a value that two or more projects use. Each of these projects links to it. No `secrets.toml` can have the name `shared` (in any letter case).
- **One copy of a value.** When a second project needs a value that one project already has, the value moves to `shared` (`secrets mv <project>/<VARIABLE> shared/<VARIABLE>`), and each project links to it. The operator decides and runs the move ([[dev-knw-nsec-rules]] § Rule 13). Never write a second copy of the value in another project.
- **Platform credentials.** The account credentials are in `shared` (`shared/VERCEL_TOKEN`, `shared/VERCEL_TEAM_ID`, `shared/RAILWAY_TOKEN`). The Convex deploy keys are in the project (`<project>/CONVEX_DEPLOY_KEY_DEV`, `…_PREVIEW`, `…_PROD`). See [[dev-knw-nsec-cli]] § The platform credentials.

## Links and implicit refs

An entry of `secrets.toml` names its secret in one of two forms:

| Form | Entry | Name |
|---|---|---|
| Implicit ref | No `ref` and no `value`: `CLERK_SECRET_KEY = { to = ["local"] }` | `<project>/<VARIABLE>` of its own project: `nuu-track/CLERK_SECRET_KEY` |
| Link | `ref = "secret://shared/<NAME>"`: `SENTRY_DSN = { ref = "secret://shared/SENTRY_DSN", to = ["local"] }` | The secret `shared/<NAME>`. The variable name of a link can be different from `<NAME>`. |

- A new variable of a project uses an implicit ref. `secrets bind <VAR> --to <targets>` with no name writes it. `bind` with a name equal to the implicit name writes the same short form.
- A `ref` points to `shared` (a link) or to the own project. A ref to the secret of another project is an error of the mapping (`BAD_LINK` in `bind`): the value moves to `shared` first, then the project links to it.
- `secrets.local.toml` accepts the same implicit refs and links. Its implicit ref names the project of the `secrets.toml` beside it.

## Declared environments

Each secret declares the environments that need a value.

- **The default set** is `dev`, `preview`, `prod`. A platform credential (the names of § Store projects and `shared`) has the default set `deploy`.
- `metadata.toml` holds only a set that is not the default of its name, under `[envs]` (for example `"nuu-track/SENTRY_AUTH_TOKEN" = ["dev", "prod"]`).
- **Missing values.** `doctor`, `pull --check`, `export`, `refs`, `plan`, and the page report a missing value only in a declared environment. A variable whose secret does not declare the environment is not delivered: `pull` and `export` add the count to their summary line (`· 1 not declared in dev`), and `plan` removes the variable from the target (`not declared in env dev`).
- **A value in an undeclared environment is not delivered.** A store file can hold such a value, for example a value that an earlier CLI wrote. `pull`, `export`, `plan`, and `push` do not use it, and they write one warning on stderr with the variable, the name, and the environments, never the value. `doctor` warns for it.
- **Writes.** `secrets set`, `secrets edit`, and the page refuse a value in an undeclared environment (`ENV_NOT_DECLARED`). `secrets env <name> <envs>` changes the set; it refuses a set that leaves a value outside it (`ENV_HAS_VALUE`). `mv` keeps the declared set of the old name. When a write removes the last value of a name, its set goes too.
- The store files and the groups do not change: `dev.enc.yaml` and `preview.enc.yaml` are in group `dev`, `prod.enc.yaml` in group `prod`, `deploy.enc.yaml` in group `deploy`.
- The operator changes the sets ([[dev-knw-nsec-rules]] § Rule 13). `secrets env <name>` (with no set) only shows the set, the environments that have a value, and the missing values: names only, no key needed.

## Tags

A tag is a label for secrets and vault entries, for example `#convex` or `#convex/deploy-keys`. Tags replace the groups by provider of the store of the previous version. A tag is not a part of the name: one secret can have many tags, and one tag can be on the secrets of many projects.

- **The grammar.** One or more segments with `/` between them. Each segment matches `[a-z0-9][a-z0-9_-]*`. At most 100 characters, and at most 32 tags on one secret or vault entry. A tag segment that looks like a secret value is refused (`BAD_TAG`).
- **The `#`.** The output shows each tag with `#`. The files hold the tags with no `#`. An argument can have the `#` or not; in a shell, put quotes around a word that starts with `#`.
- **Children.** A tag includes its children: `--tag convex` selects `#convex`, `#convex/deploy-keys`, and each other child of `#convex`, not `#convexity`.
- **Files.** The tags of the secrets and of the vault entries are in `metadata.toml` (`[tags]` and `[vault-tags]`). A tag can have a record in `tags.toml`: `description` (one line, at most 200 characters), `url` (an `https://` URL with no user name and no password), and `warn-days` (1 to 365). The secret scan of each store commit reads both files: no secret goes in a tag, a description, or a URL.
- **Reads.** `secrets tag list` shows the tag tree with the counts, the windows, and the records. `--tag <tag>` on `secrets list`, `secrets expiry list`, `secrets vault list`, and `secrets log` selects the tag and its children. They read plaintext files only and need no key.
- **The tags follow the writes.** `mv` and `vault mv` move the tags with the name. When a write removes the last value of a secret, its tags go too.
- The operator changes the tags and the records ([[dev-knw-nsec-rules]] § Rule 13). When you select a tag for a report or a question, use an existing tag of `secrets tag list`.

## A store of the previous version

A store of the previous version has lowercase path names (for example `clerk/secret-key`), and it can have a plaintext file with the records of its groups by provider. The CLI still reads the old names (`secrets list`, `doctor`, `log`), and `doctor` warns for them. A store of the previous version has no `store.toml` (the store format 1). The v2 CLI refuses its reads and its writes (`STORE_FORMAT`), and only `doctor`, `whoami`, `store list`, `store sync`, `store clone`, `store remove`, and `store migrate` still run on it. A store of the previous version needs the migration: the guide is [[dev-knw-nsec-migrate_v2]]. The migration is an operator write ([[dev-knw-nsec-rules]] § Rule 13).

## Expiry dates

A secret can have one expiry date in each environment: the first day on which its key does not work. The prod key and the dev key are often different keys. The dates are in `metadata.toml` (plaintext, no values).

- **The state** on a day: `expired` from the date on; `due` when the date is inside the warning window; else `ok`. The window is the `warn-days` of the entry. Else, for each tag of the secret, the `warn-days` of the nearest tag record that has one (the record of the tag, else of its parent, and so on up); with two or more tags, the largest of these windows. Else 14 days. The day is the local date of this machine.
- **Warnings, never errors.** The date is the estimate of a person. `pull` (also `pull --check`) and `export` write one stderr line for each mapped secret of the environment that is `due` or `expired`. For `preview`, the date of the preview value. `doctor` shows each one as a `warn` line. `secrets store sync` lists the `due` and `expired` secrets of the store.
- **The other writes keep the dates right.** `unset`, `edit`, and the page remove the date of a removed name. `mv` moves the dates with the name. A new value does not change the date: the operator sets the date of the new key (`secrets set <name> --env <env> --expires <date>`, or `secrets expiry set`).
- **The same value, the same date.** A write of a date (`expiry set`, `expiry clear`, `set --expires`) also changes the date of each other place with the same plaintext inside the recipient group of the value (`dev` with `preview`, `prod` alone, `deploy` alone). `--all-groups` compares each group that the machine opens, and `--only` changes the one value. The compare reads values on the machine of the operator and writes names only.
- The operator sets and clears the dates. You report a warning ([[dev-knw-nsec-rules]] § Rule 13).

## The change log and moved refs

Each store write adds its changes to `changes.jsonl` in the same commit: `add`, `change`, `remove`, and `move` (a secret name, with its environments), `tag-add`, `tag-remove`, `tag-rename`, `tag-record-change`, `env-set-change`, `expiry-set|clear`, and `vault-add|change|remove|move` (a vault entry name and its group). Each line has the time (UTC) and `by`: the recipient id of the key of the write, or `null`. A line never holds a value or a vault text. `secrets log` shows the log, newest first. `secrets store sync` shows the lines since the last sync of this machine.

- Recipient changes (`recipient add|rm`) and `store fix-config` add no line: `secrets store sync` shows the recipient changes from `recipients.toml`.
- A store of the previous version can also have old entries of its groups by provider: they stay valid, and `secrets log` shows them.
- A store of an earlier version has no `changes.jsonl`. A write of an earlier version of the CLI adds no line.

**The trace of a ref.** When a `ref` of `secrets.toml` or `secrets.local.toml` has no value, the CLI follows its name through the change log: each later `move` to the new name, until a name that a store file holds, a `remove`, or the end of the log. A name that a later write added again is a new secret: then the log says nothing about it.

| State (`secrets refs`) | Meaning |
|---|---|
| `ok` | A store file holds the name. |
| moved | The log moved the name: `moved to secret://<new name> on <date> by <recipient id>`. The CLI never uses the value of the new name. The project file must change first (`secrets refs fix`, with the approval of the operator). |
| removed | The log removed the name: `removed on <date> by <recipient id>`. The operator decides what the entry gets. |
| not in the store | No store file holds the name, and the log has no line for it: a missing value. |

- The error of a missing value (`pull`, `export`) and the `doctor` line add the hint. An `optional` entry with a moved or removed name gives the same hint as a warning.
- `plan` and `push` stop on a moved ref, also on an `optional` ref: a push never removes the variable of the platform because a secret has a new name.
- `secrets refs fix` changes only the text of the ref of each entry, and keeps every other byte of the file. It refuses a project file with a symbolic link in its path.

## The vault

The vault keeps free text that is not an environment variable: passwords, logins, recovery codes, and private keys. Agents never read or write a vault text ([[dev-knw-nsec-rules]] § Rule 14).

- **Not part of the environments.** No `secrets.toml` maps a vault entry. `pull`, `export`, `plan`, and `push` never read the vault, and `edit`, `mv`, `unset`, and `secrets tag` never change it. `secrets vault tag add|rm` changes the tags of an entry (in `metadata.toml`), never a vault file.
- **Files.** One entry is one file at `vault/<group>/<name>.enc.yaml`, with one key, `text`. The entry name is a lowercase path (`[a-z0-9][a-z0-9_-]*` for each segment): vault entries are outside the store projects. It is plaintext (the path of the file), and it is in one group only. The text is always encrypted, for exactly the recipients of the group in `recipients.toml`. No other file can be under `vault/`.
- **Groups.** `dev` or `prod` (the default is `prod`). The machines of that group open the entry. `deploy` is not a vault group: CI machines do not read the vault.
- **The text.** Free UTF-8 text, not empty, at most 64 KiB, with no NUL character. It comes back byte for byte.
- **Writes.** The operator writes the vault: `vault edit`, `vault set`, `vault mv`, `vault rm`, or the Vault view of `secrets ui`. Each write is one store commit through the write path of the CLI. A machine changes only the entries of its own groups. When another machine changed the same entry, the write stops with a conflict (exit 3).
- **Recipients.** `recipient add` and `recipient rm` encrypt the vault entries of the changed groups again. `recipient rm --rotate` lists the vault entries that the removed key could open: the operator changes each password or key at its service, then saves the new text.
- **The checks.** Each store write checks the vault paths and the policy of each vault file first, with no decrypt. A broken vault file stops each write of its store, also `secrets set` (`UNSAFE_VAULT` or `UNSAFE_STORE`). `doctor` shows each problem in lines that start with `vault:`.

## The environments

`--env` selects the environment. The default is `dev`. Each environment reads only its own store file.

| `--env` | File that the CLI reads | Group that this machine needs |
|---|---|---|
| `dev` | `dev.enc.yaml` | `dev` |
| `preview` | `preview.enc.yaml` only | `dev` |
| `prod` | `prod.enc.yaml` only | `prod` |
| `deploy` | `deploy.enc.yaml` only | `deploy` |

- `preview` does not read `dev`. A secret that declares `preview` needs its own value in `preview.enc.yaml`. A required entry with no preview value is an error in `pull --env preview`, `export --env preview`, `plan`, and `push`, also when dev has a value. The operator writes the value (`secrets set <name> --env preview`).
- `prod` does not read `dev`. A name that is only in `dev.enc.yaml` has no value in `prod`.
- `deploy` holds platform credentials.
- An environment that the secret does not declare gives no value (§ Declared environments).

## The project mapping: `secrets.toml`

Each project has one committed `secrets.toml`. It holds names and targets, never values.

```toml
[project]
name = "nuu-track"
store = "git@github.com:NUU-Cognition/nuu-secrets.git"   # a Git URL or a local path

[targets.vercel]
project = "nuu-track"

[env]
CLERK_SECRET_KEY = { to = ["local", "vercel"] }                                             # implicit: secret://nuu-track/CLERK_SECRET_KEY
SENTRY_DSN       = { ref = "secret://shared/SENTRY_DSN", to = ["local"], optional = true }   # a link to shared
LOG_LEVEL        = { value = "info", to = ["local"] }
```

| Part | Rule |
|---|---|
| `[project]` | Two fields only: `name` (a letter or a digit first, then letters, digits, `.`, `_`, or `-`; 100 characters or fewer) and `store` (the Git URL or the path of the store). The URL must not hold a credential: `secrets init-project` refuses such a URL. |
| `[targets.<target>]` | One table for each platform target: `vercel`, `convex`, or `railway`. See [The target tables](#the-target-tables). |
| `[env]` | One entry for each variable. The key is the variable name: letters, digits, and `_`, and not a digit first. |
| `ref` | A reference `secret://<name>` to a value in the store: a link to `shared`, or a name of the own project. With no `ref` and no `value`, the entry is an implicit ref to `<project>/<VARIABLE>`. |
| `value` | A literal: a value that is not secret. It is committed in plaintext. |
| `to` | A list of targets, without repeats: `local`, `vercel`, `convex`, `railway`. |
| `optional` | `true` or `false`. Only for a store entry (a `ref` or an implicit ref). |

- Each entry has at most one of `ref` and `value`. The fields are `ref`, `value`, `to`, and `optional`. Another field is an error.
- `local` is the target of `secrets pull` (the block of `.env.local`) and of `secrets export` (the default `--to local`). A platform target needs its `[targets.<target>]` table.
- When the environment has no value for an `optional` entry, the CLI does not give that variable and does not stop. The block does not have it, and the `pull` line counts it as "optional not set". The mapping still owns the name: a line outside the block that sets it stops the pull.
- A store entry with no value in a declared environment, and not `optional`, is an error. The error names the variable and the secret name. Nothing is written.
- The CLI finds `secrets.toml` in the current folder or in a parent folder. `--mapping <path>` gives another file. `--project <name>` finds the `secrets.toml` with that project name under the root of the current Git repository. It skips `node_modules`, `.git`, and `dist*`.
- The `store` of the mapping selects the store. When this machine has no clone of that store, `secrets pull` clones it (a network access) and writes `secrets: cloning the store of <project> from <url>` on stderr. With `--offline`, it never clones: it stops with exit 1.

## The target tables

`secrets plan` and `secrets push` read the `[targets.<target>]` table of each platform target. The CLI checks each table when it loads the mapping.

```toml
[targets.vercel]
project = "nuu-track"                    # a Vercel project name or id

[targets.convex]
deployment = "track"                     # a label for the output: the deploy key is nuu-track/CONVEX_DEPLOY_KEY_<ENV>

[targets.railway]
project      = "nuu-discord-bot"         # a Railway project name or id
service      = "bot"                     # a Railway service name or id
environments = { prod = "production" }   # store environment → Railway environment name or id
```

- `[targets.vercel]` has `project` only. The team id comes from `shared/VERCEL_TEAM_ID` in `deploy.enc.yaml`: a `team` field is an error.
- `[targets.convex]` has `deployment`: one name segment (lowercase letters, digits, `_`, and `-`). It is a label for the output only: the deploy key of each environment is `<project>/CONVEX_DEPLOY_KEY_<ENV>`.
- `[targets.railway]` has `project`, `service`, and `environments` (at least one entry; the keys are `dev`, `preview`, and `prod`). An environment with no entry cannot go to Railway.
- A missing field, an unknown field, or a bad name is an error of the mapping. A table that is not valid stops **every** command of that project, also `pull` and `export`. Run `secrets doctor`: it names each problem and never shows a value.
- The Convex tool owns some names, and the CLI never writes them to a Convex deployment: `CONVEX_DEPLOYMENT`, `CONVEX_DEPLOY_KEY`, `CONVEX_DEPLOYMENT_TOKEN`, `CONVEX_SELF_HOSTED_URL`, `CONVEX_SELF_HOSTED_ADMIN_KEY`, `CONVEX_CLOUD_URL`, `CONVEX_URL`, `CONVEX_SITE_URL`, and the client URL names (`NEXT_PUBLIC_CONVEX_URL`, `VITE_CONVEX_URL`, `PUBLIC_CONVEX_URL`, `REACT_APP_CONVEX_URL`, `EXPO_PUBLIC_CONVEX_URL`, and the same `…_CONVEX_SITE_URL` names). An entry with one of these names cannot have the target `convex`. The CLI accepts another target (for example `NEXT_PUBLIC_CONVEX_URL` to `vercel`), but that is a decision of the operator: for local development, `npx convex dev` writes `CONVEX_DEPLOYMENT` and `NEXT_PUBLIC_CONVEX_URL` to `.env.local` itself, outside the block. A name with the target `local` that the Convex tool also writes stops each pull (a managed name outside the block): the usual decision of the operator is to remove `local` from that entry.
- A platform target and its table are operator decisions ([[dev-knw-nsec-rules]] § Rule 8). The delivery is in [[dev-sk-nsec-plan_push]].

## The local mapping: `secrets.local.toml`

`secrets.local.toml` is an optional file beside `secrets.toml`. It is for this machine only.

- It must be in `.gitignore`. `secrets doctor` fails when Git tracks it.
- It can hold only an `[env]` table. Each entry can target only `local`.
- For the `local` target, its entries replace the entries of `secrets.toml` with the same variable.
- `secrets export --to <platform target>` does not use its entries in the output. But the CLI still reads and checks the file: a `secrets.local.toml` that is not valid stops that export too (exit 1).
- `secrets bind` writes `secrets.toml`. When `secrets.local.toml` has the same variable, `bind` writes a notice on stderr: the local entry wins on this machine.
- `secrets bind … --local` writes an entry to `secrets.local.toml` (it makes the file when it is missing), and `secrets unbind <VAR> --local` removes one. `bind --local` refuses when Git does not ignore the file or tracks it. Do not edit the file by hand.

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
| A secret: an API key, a token, a password, a signing key, a URL with a credential | The store, in the project: an implicit ref. In `shared` when two or more projects use the value: a link. |
| A value that is not secret, but is different in each environment | The store, as above. |
| A value that is not secret and is the same in all environments | A literal in `secrets.toml`. |
| A different entry for this machine only | An entry in `secrets.local.toml`: an implicit ref, a link to `shared`, or a literal that is not secret. Never a value. |
| A password, a login, recovery codes, or a private key that no program reads from its environment | The vault. The operator writes it. Never a `secrets.toml` entry, also not a literal. |
| A variable that another tool writes and manages, for example `CONVEX_DEPLOYMENT` and `NEXT_PUBLIC_CONVEX_URL` (written by `npx convex dev` into `.env.local`) | Not in the store and not in the target `local` of the mapping. The tool owns its line, outside the block. A push never sends it to Convex. Do not open or change `.env.local`. |

- A `NEXT_PUBLIC_*` variable is for the browser build: it is public in the bundle. `secrets doctor` warns when a `NEXT_PUBLIC_*` entry targets `convex`.
- A variable of a `.env*` file that the mapping also manages is a duplicate. `secrets doctor` warns and names it. The operator removes it from the `.env*` file: `secrets pull` writes the variable into the block of `.env.local`.

## The block of `.env.local` (`secrets pull`)

- `secrets pull` resolves the variables of the target `local` (with `secrets.local.toml`) for `dev` or `preview`, and writes them into one block of the `.env.local` in the folder of the `secrets.toml`. `prod` and `deploy` never go to the disk.
- The block is the lines between the start marker `# >>> nuu-secrets · store <revision> (<date>) · env <env> · sha256 <hash of the lines> · do not edit: change the store, then run `secrets pull` >>>` and the end marker `# <<< nuu-secrets <<<`. The CLI owns these lines. Each line outside the block belongs to the person or to another tool, and stays byte for byte.
- The direction is one way: store → `.env.local`. A hand change of the block makes the hash wrong, and the next pull stops (`PULL_CHANGED_BLOCK`; `--force` replaces the block). To change a value, the operator changes the store, then a pull writes the new block.
- A managed name (a variable that the mapping gives to `local`) belongs only in the block. A line outside the block that sets a managed name stops each pull (`PULL_MANAGED_OUTSIDE`). The pull never removes the line. The operator decides: remove the line, or remove the target `local` of the name.
- `.env.local` is the one file with plaintext values that the CLI writes. It has mode 600, and Git must ignore it and must not track it: else the pull stops. The CLI never writes through a symbolic link.
- The program then reads `.env.local` with its own loader (Next.js, Vite, the Convex CLI, `node --env-file=.env.local`). A reader that expands `$` (dotenv-expand, for example in Next.js and Vite) can change a value with `$`.
- Other processes of the same user, a backup, and a sync tool can read `.env.local`. On macOS, a project folder in iCloud Drive gives a warning: iCloud keeps a copy.
- A refresh or a first clone also changes the Git files and the index of the store clone in the secrets home (ciphertext and public data only).
- Agents never read, print, copy, or change the block, and never open `.env.local` ([[dev-knw-nsec-rules]] § Rule 15). Agents never make another file with values, for example with a redirect of `secrets export` ([[dev-knw-nsec-rules]] § Rule 2).
