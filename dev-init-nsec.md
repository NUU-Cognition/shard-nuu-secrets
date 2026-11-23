---
required-reading:
  - "[[dev-knw-nsec-rules]]"
---

# NUU Secrets (nsec)

This shard teaches agents to use NUU Secrets safely. With it, you run apps with their secrets, map the variables of a project, and find problems. You never see, write, or move a value or a vault text.

## What NUU Secrets Is

NUU Secrets keeps the secrets of a team in one encrypted Git store and gives them to programs. The CLI is `secrets`. `secrets pull` writes the values of a project into one block of its `.env.local`. Then the program runs with its normal command.

- **One store for each team.** The store is a private Git repository that the team owns. SOPS encrypts each value. The secret names, the recipients, and the policy metadata are plaintext. Age keys open the files. Each machine has one age key. For NUU Cognition, the store is `NUU-Cognition/nuu-secrets`.
- **Four environments.** `dev`, `preview`, `prod`, and `deploy`. Each environment has one store file, and each environment reads only its own file: `preview` does not read `dev`, and `prod` does not read `dev`. `deploy` holds platform credentials.
- **Names by project.** A secret name is `<project>/<VARIABLE>`, for example `nuu-track/CLERK_SECRET_KEY`. The project is the `[project] name` of one `secrets.toml`, or `shared`. A value that two or more projects use has one copy, in `shared`, and each project links to it. `secrets list` shows the names by project.
- **Declared environments.** Each secret declares the environments that need a value: by default `dev`, `preview`, `prod`; a platform credential has `deploy`. A value in an environment that the secret does not declare is not delivered. `secrets env <name>` shows the set.
- **Groups.** A machine opens only the files of its groups (`dev`, `prod`, `deploy`). The group `dev` opens `dev` and `preview`.
- **One `secrets.toml` for each project.** It is committed. It maps each variable to a secret: an implicit ref (`CLERK_SECRET_KEY = { to = ["local"] }` names `<project>/CLERK_SECRET_KEY`), a link to `shared` (`SENTRY_DSN = { ref = "secret://shared/SENTRY_DSN", to = ["local"] }`), or a literal that is not secret. It holds no values.
- **`secrets pull`.** It reads the store and writes the values of the target `local` (`dev` or `preview`, never `prod` or `deploy`) into one block of the `.env.local` in the folder of the `secrets.toml`. The block is the lines between `# >>> nuu-secrets · …` and `# <<< nuu-secrets <<<`. The CLI owns the block. Each line outside the block stays byte for byte. The direction is one way: to change a value, change the store, then pull again. `secrets pull --check` writes nothing and says whether the block is current. Then you run the normal dev command (`pnpm dev`, `next dev`, `vite`, `npx convex dev`): it reads `.env.local`. `.env.local` is the one file with plaintext values that the CLI writes. A refresh or a first clone also changes the Git files and the index of the store clone. Since `0.4.0-beta.0`, the CLI has no `run` command and no short alias that starts a program: an old instruction that names them gives `error: unknown command 'run'`.
- **Delivery to a platform.** `secrets plan <target> --env <env>` shows what a push sends to Vercel, Convex, or Railway: names and statuses only. `secrets push` sends the values. Only the operator runs `push`.
- **The operator.** Only the operator enters values (`secrets set`), manages keys, and pushes to a platform (`secrets push`). You ask the operator for them.
- **The secret scan.** Before a store commit, the CLI scans the exact bytes of the change unless gitleaks is missing or `SECRETS_GITLEAKS=0`. A finding or a failed scan stops the write. Only the operator decides that a finding is a false positive.
- **No account.** A NUU account is optional. No command needs a login. NUU hosts no secrets.
- **The setup.** Each setup step is a command of the CLI: no step needs a hand edit of a file. `secrets setup` guides the operator through all steps on a terminal (an operator command). `secrets setup --print` shows the steps that remain and their commands, and changes nothing.
- **Identities.** An identity is a name for one age key file of this machine (`identities.toml`). One machine can have several identities, for example one for the store of the team and one for the store of a client. For each command, the CLI selects one key file: `--identity`, `SECRETS_IDENTITY`, `SOPS_AGE_KEY_FILE`, the identity of the project in `projects.toml`, the default identity, then `~/.config/sops/age/keys.txt`. You use the identity that the CLI selects.
- **The project index.** `projects.toml` lists the projects of this machine: paths, names, stores, and identities, never values. Each command that loads a `secrets.toml` records its project. `secrets project add <folder>` adds the projects under a folder. `secrets project list` shows the index.
- **Tags.** A tag is a hierarchical label of secrets and vault entries, for example `#convex/deploy-keys`. A tag includes its children. `tags.toml` can give a tag a description, a source URL, and a warning window. `secrets tag list` shows the tag tree, and `--tag <tag>` selects the secrets of a tag.
- **A store of the previous version.** It has lowercase path names (for example `clerk/secret-key`). It has the store format 1 (no `store.toml`), and the v2 CLI refuses its reads and its writes (`STORE_FORMAT`): `doctor` shows `fail  store format 1 …`. A store of the previous version needs the migration: the guide is [[dev-knw-nsec-migrate_v2]]. The migration is an operator write. [[dev-sk-nsec-migrate_v2]] runs only the read-only steps and gives the operator the write commands.
- **Expiry dates.** The operator can give a secret one expiry date in each environment (`metadata.toml`). `secrets pull`, `export`, `doctor`, and `store sync` warn when a date is near (`due`) or past (`expired`). `secrets expiry list` shows the dates.
- **The change log.** Each store write adds its changes to `changes.jsonl` in the store: names, environments, dates, and the recipient id of the writer, never a value. `secrets log` shows it. When a ref has no value, the CLI follows the change log: the error says `moved to secret://<new name>` or `removed`. `secrets refs` shows the state of each ref, and `secrets refs fix` rewrites the moved refs (with the approval of the operator).
- **The vault.** Free text that is not an environment variable: passwords, logins, recovery codes, private keys. Each vault entry is encrypted for the machines of one group (`dev` or `prod`). The vault is not part of the environments: `secrets pull` never writes a vault entry to `.env.local`. You never read or write a vault text. `secrets vault list` shows the names only.
- **The UI.** `secrets ui` is a local web page for the operator: the stores, the projects, the store projects, the change log, the vault, and the values and vault texts (hidden until a reveal). The operator can set values and vault texts there. You never run it and never read a value or a vault text from it.

The full model is in [[dev-knw-nsec-model]]. The command reference, the exit statuses, and the known limits are in [[dev-knw-nsec-cli]]. The setup of a new team, a new machine, and a project is in [[dev-knw-nsec-setup]]. The migration of a store of the previous version is in [[dev-knw-nsec-migrate_v2]].

## The Rules in Short

The complete rules are in [[dev-knw-nsec-rules]]. Read them before your first `secrets` command.

1. Never print, log, echo, or type a value. Not in output, a commit message, a Mesh note, a message, a test fixture, or a command.
2. Never write a value to a file. No `.env` file, no `secrets export … > file`. Only `secrets pull` writes values to a file: the block of the `.env.local` of a project (see 15). If another tool needs a file with values, stop and ask the operator.
3. Never pass a value as a process argument. A program reads a value from `.env.local` or from its process environment.
4. Never read or copy a key file (the key file of each identity, `$SOPS_AGE_KEY_FILE`, `~/.config/sops/age/keys.txt`, and `<secrets home>/keys/`). For the public key, run `secrets keygen --public`. Never decrypt a store file or a vault file. When a read stops with `… has SOPS metadata that the CLI does not decrypt. …`, stop and give the operator the exact message. Never edit `*.enc.yaml`, `.sops.yaml`, `.gitleaks.toml`, `recipients.toml`, `tags.toml`, `metadata.toml`, `changes.jsonl`, `identities.toml`, or `projects.toml`.
5. Never run `secrets setup` (except `setup --print`), `set`, `edit`, `unset`, `keygen` (except `keygen --public` with no `--file`), `recipient add|rm`, `store init`, `store fix-config`, `store remove`, `push`, `login`, `logout`, `identity add|use|rm`, `project identity <name>`, `project identity --clear`, `project forget`, `mv`, `env <name> <envs>`, `tag add|rm|edit|rename`, `vault tag add|rm`, `expiry set|clear`, `refs fix` (except `refs fix --dry-run`), `vault mv|rm`, or `pull --force`, unless the operator asks for or approves that exact command. Never add `--yes` to a push.
6. Ask the operator for a missing value with the exact command: `secrets set <name> --env <env>`, or the page of `secrets ui`. Then continue with `secrets pull`. Never ask for a value in the chat. A password or a login from the vault: see 14. A moved or removed ref: see 13.
7. You are a user. You have the access of your operating-system user. The rules prevent accidents. They are not a security boundary.
8. If a value leaks, stop, do not repeat it, and tell the operator the variable and the place. The operator replaces the value.
9. Never set `SECRETS_GITLEAKS=0`, and never remove, rename, or hide gitleaks on `PATH`. Before a store write that this shard allows, check for an inherited `SECRETS_GITLEAKS=0` and report it. When the secret scan stops a write (`SECRET_SCAN_FOUND`, `SECRET_SCAN_FAILED`), stop and report. See [[dev-knw-nsec-rules]] § Rule 10.
10. Change `secrets.toml`, `secrets.local.toml`, and the NUU Secrets lines of `.gitignore` only with the commands of the CLI (`init-project`, `bind`, `unbind`, `target set|rm`, `project gitignore`, `project set`, and `refs fix` with the approval of the operator), not by hand. Write a target table or change the store URL only with the ids or the URL that the operator gives.
11. Never run `secrets ui`, also when the operator asks. Never open its URL or login link, its page, or its API, and never read a value or a vault text from it. The UI is for the operator. See [[dev-knw-nsec-rules]] § Rule 11.
12. Use the identity that the CLI selects. Do not add `--identity`, and do not set `SECRETS_IDENTITY` or `SOPS_AGE_KEY_FILE`, unless the operator names the identity. When a command or `doctor` reports `LINKED_MAPPING`, `MAPPING_CHANGED`, `OUTSIDE_PROJECT`, `DUPLICATE_PROJECT`, an unknown identity, `PROJECT_INDEX`, the link warning, or the warning about the identity of a parent project, stop and tell the operator the exact message and the remedy of the CLI. See [[dev-knw-nsec-rules]] § Rule 12.
13. Moves to `shared`, tag changes, environment sets, and the store migration are operator writes. No agent migrates a store or the secrets of a local repository. When a project needs a value that another project has, tell the operator: the operator decides and runs the move to `shared`. A value in an undeclared environment is not delivered: report the warning. A missing ref with a `moved` hint: run `secrets refs fix --dry-run` and ask the operator to approve the fix. A `removed` ref: ask the operator what to do. An expiry warning (`due` or `expired`): tell the operator; never run `expiry set|clear` yourself. See [[dev-knw-nsec-rules]] § Rule 13.
14. Never read or write a vault text. Never run `secrets vault show` (also `--copy`), `secrets vault edit`, or `secrets vault set`, also when the operator asks: the operator runs them in a terminal. Never read the clipboard. When a task needs a password, a login, or another text from the vault, ask the operator to do that step. `secrets vault list` is allowed (names only). `secrets vault mv` and `secrets vault rm` need the approval of the operator. The vault is not part of the environments: never tell the operator to put a value of an environment into the vault, or a vault text into `secrets.toml`. See [[dev-knw-nsec-rules]] § Rule 14.
15. Never read, print, copy, or change the values in the block of `.env.local`, and never open or search that file. Change a value in the store (the operator, with `secrets set`), never in `.env.local`. Before you run an app, run `secrets pull --check`, and `secrets pull` when the block is not current. Never add `--force` without the approval of the operator. When `secrets pull` stops on a managed name outside the block, report each line number and name to the operator. Do not delete or change the line without the approval of the operator. See [[dev-knw-nsec-rules]] § Rule 15.

## Situations

| Situation | Use |
|---|---|
| Run an app, a test, or a script with its secrets | [[dev-sk-nsec-run_app]]: `secrets pull --check`, `secrets pull` when the block is not current, then the normal command |
| `secrets pull` stops on a name outside the block, a changed block, a broken block, or a `.env.local` that Git does not ignore or tracks | [[dev-sk-nsec-run_app]] (step 7). Report a managed name outside the block to the operator; never delete the line without the approval of the operator |
| An old instruction, script, or note starts a program through the CLI (the old `run` command or its alias), or the CLI says `error: unknown command 'run'` | [[dev-sk-nsec-run_app]]: pull, then the normal command |
| A project has no `secrets.toml`, or a variable has no entry | [[dev-sk-nsec-setup_project]] |
| Help an operator set up NUU Secrets: a new team (a new store), a new machine, or the first project | [[dev-knw-nsec-setup]]; run `secrets setup --print` first |
| The shell cannot find `secrets` | [[dev-knw-nsec-cli]] § Install: ask the operator to install the CLI |
| `secrets` fails, or access is not correct | [[dev-sk-nsec-diagnose]] |
| A store write stops with the secret scan, or `doctor` warns about gitleaks | [[dev-sk-nsec-diagnose]] (step 10) and [[dev-knw-nsec-rules]] § Rule 10 |
| `doctor` warns `.gitleaks.toml differs from the text that the CLI generates` | [[dev-sk-nsec-diagnose]] (step 4): give the operator the exact `secrets store fix-config --store <name>` command; never run it |
| A read stops with `✘ The store file <file> of store <store> has SOPS metadata that the CLI does not decrypt. SOPS did not run, and nothing was read.` (`UNSAFE_STORE`), or with the same message for a vault file (`UNSAFE_VAULT`) | Stop. Do not touch the file. [[dev-sk-nsec-diagnose]] (step 4): give the operator the exact message |
| A value or a key is missing | [[dev-sk-nsec-request_value]] |
| A command or `doctor` reports an identity problem (`LINKED_MAPPING`, `MAPPING_CHANGED`, `OUTSIDE_PROJECT`, `DUPLICATE_PROJECT`, an unknown identity, a parent-identity warning) | Stop. [[dev-sk-nsec-diagnose]] (step 6) and [[dev-knw-nsec-rules]] § Rule 12 |
| The operator asks about identities, the project index, or `secrets ui` | [[dev-knw-nsec-setup]] § Identities, § The project index, § The UI; give the operator the commands |
| The task asks you to use `secrets ui`, or the operator sends a link `http://127.0.0.1:<port>/#t=…` | Do not run or open it. [[dev-knw-nsec-rules]] § Rule 11 |
| Prepare a push to Vercel, Convex, or Railway | [[dev-sk-nsec-plan_push]] |
| A missing ref says `moved to secret://…` or `removed on …` (`pull`, `doctor`, `plan`, or `secrets refs`) | [[dev-sk-nsec-diagnose]] (step 7): `secrets refs fix --dry-run`, then ask the operator |
| stderr or `doctor` has an expiry warning (`expires on …` or `expired on …`) | Tell the operator the variable, the secret name, the environment, and the date. [[dev-knw-nsec-rules]] § Rule 13 |
| A task needs a password, a login, recovery codes, or a private key from the vault | Do not read it. Ask the operator to do the step that needs it ([[dev-sk-nsec-request_value]], step 5). [[dev-knw-nsec-rules]] § Rule 14 |
| The task asks you to run `secrets vault show`, `vault edit`, or `vault set` | Do not run it, also when the operator asks. Give the operator the command for their own terminal. [[dev-knw-nsec-rules]] § Rule 14 |
| A project needs a value that another project already has (`secrets list` shows it there) | Do not write a copy and do not link to the other project. Tell the operator: the operator decides and runs the move to `shared`. [[dev-sk-nsec-setup_project]] (step 6), [[dev-knw-nsec-rules]] § Rule 13 |
| A warning says that a value of an environment is not used (`… does not declare env <env>`), or `preview` has no value while `dev` has one | Report it to the operator with the commands of the warning; do not run them. [[dev-knw-nsec-rules]] § Rule 13 |
| `doctor` warns for names of the previous version, or the task asks you to migrate a store or the secrets of a repository | Do not migrate. Report it: the migration is an operator write ([[dev-knw-nsec-migrate_v2]]). When the operator asks for help with the migration: [[dev-sk-nsec-migrate_v2]]. [[dev-knw-nsec-rules]] § Rule 13 |
| The operator asks about tags, environment sets, expiry dates, the change log, or the vault | [[dev-knw-nsec-setup]] § Tags, environment sets, expiry dates, and the change log, and § The vault; give the operator the commands |
| A command stops with `This store has format 1` (`STORE_FORMAT`), or `doctor` shows `fail  store format 1 …` | The store needs the migration (an operator write). Do not migrate. Tell the operator, with the path of [[dev-knw-nsec-migrate_v2]]. When the operator asks for help with the migration: [[dev-sk-nsec-migrate_v2]] (you run only its read-only steps) |
| A question about the store, names, `shared`, links, declared environments, tags, targets, or literals | [[dev-knw-nsec-model]] |
| A question about a command, an option, an exit status, or a message | [[dev-knw-nsec-cli]] |

## Lifecycle

The usual order of the work in one project:

```
setup_project → request_value → (operator: secrets set or secrets ui) → run_app (pull, then the dev command)
                      ↑                                        │
                      └──────────── diagnose ←── a failure ────┘
```

## Commands at a Glance

```sh
secrets pull --check                     # is the block of .env.local current? writes nothing
secrets pull                             # write the dev values into the block of .env.local
secrets pull --env preview               # the preview values (preview.enc.yaml only); never prod or deploy
secrets pull --all                       # each project of the index in this Git repository
pnpm dev                                 # then the normal command; it reads .env.local
secrets doctor                           # check everything offline; changes nothing
secrets doctor --online                  # after a plain doctor with the same options: refresh (fetch and reset), then check
secrets whoami                           # the id, the identity, the key fingerprint, the stores, the groups
secrets identity list                    # the identities, their public keys and stores, and the active selection
secrets project list                     # the project index: name, store, identity, state, path
secrets project identity                 # show the identity of this project; changes nothing
secrets project add <folder> --dry-run   # find each secrets.toml under the folder; without --dry-run, add to projects.toml
secrets store list                       # the store clones of this machine
secrets log                              # the change log of the store: names, dates, writers; no values
secrets refs                             # the state of each ref: ok, moved, removed, not in the store
secrets refs fix --dry-run               # the plan of the ref fix; without --dry-run: only with the approval of the operator
secrets list                             # the secret names by project, their environments, tags, and users
secrets list --project shared            # the shared secrets and the projects that link to them
secrets env <name>                       # the declared environments of one secret; with a set: operator only
secrets tag list                         # the tag tree with the counts and the records
secrets expiry list --due                # the secrets that are due or expired
secrets vault list                       # the vault entries: names, groups, access; never a text
secrets keygen --public                  # the public key of this machine; makes no key
secrets setup --print                    # the state of each setup step and the commands that remain; changes nothing
secrets init-project --name <project> --store <url>   # also adds .env* and secrets.local.toml to .gitignore
secrets vars --unmapped                  # the variable names that the code reads and no mapping has
secrets bind <VAR> --to local            # the implicit ref <project>/<VAR>
secrets bind <VAR> shared/<NAME> --to local   # a link to a shared value
secrets unbind <VAR>                     # remove one entry
secrets target set vercel --project <id> # a target table, with the ids from the operator
secrets project gitignore                # add the missing .gitignore lines
secrets plan <target> --env <env>        # what a push sends: names and statuses; no platform call
```

Use the `secrets` command that is on `PATH`. Its build mode (prod or dev) changes the default secrets home. See [[dev-knw-nsec-model]] § The store.

When the shell cannot find `secrets`, do not install it yourself. Ask the operator to install the CLI: `npm install -g @nuucognition/secrets-cli@0.4.0-beta.0` (Node.js 24 or later). See [[dev-knw-nsec-cli]] § Install.
