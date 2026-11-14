---
required-reading:
  - "[[dev-knw-nsec-rules]]"
---

# NUU Secrets (nsec)

This shard teaches agents to use NUU Secrets safely. With it, you run apps with their secrets, map the variables of a project, and find problems. You never see, write, or move a value.

## What NUU Secrets Is

NUU Secrets keeps the secrets of a team in one encrypted Git store and gives them to programs. The CLI is `secrets`. Its alias `secret` runs a program with the values.

- **One store for each team.** The store is a private Git repository that the team owns. SOPS encrypts each value. The secret names, the recipients, and the policy metadata are plaintext. Age keys open the files. Each machine has one age key. For NUU Cognition, the store is `NUU-Cognition/nuu-secrets`.
- **Four environments.** `dev`, `preview`, `prod`, and `deploy`. Each environment has one store file. `preview` reads `dev` and then its own values over it. `prod` never falls back to `dev`. `deploy` holds platform credentials.
- **Groups.** A machine opens only the files of its groups (`dev`, `prod`, `deploy`). The group `dev` opens `dev` and `preview`.
- **One `secrets.toml` for each project.** It is committed. It maps each variable to a secret name (`CLERK_SECRET_KEY = { ref = "secret://clerk/secret-key", to = ["local"] }`) or to a literal that is not secret. It holds no values.
- **`secret <cmd>`.** It reads the store, decrypts in memory, and starts `<cmd>` with the values in its process environment. It writes no file with a plaintext value. A refresh or a first clone changes the Git files and the index of the store clone.
- **Delivery to a platform.** `secrets plan <target> --env <env>` shows what a push sends to Vercel, Convex, or Railway: names and statuses only. `secrets push` sends the values. Only the operator runs `push`.
- **The operator.** Only the operator enters values (`secrets set`), manages keys, and pushes to a platform (`secrets push`). You ask the operator for them.
- **The secret scan.** Before a store commit, the CLI scans the exact bytes of the change unless gitleaks is missing or `SECRETS_GITLEAKS=0`. A finding or a failed scan stops the write. Only the operator decides that a finding is a false positive.
- **No account.** A NUU account is optional. No command needs a login. NUU hosts no secrets.
- **The setup.** Each setup step is a command of the CLI: no step needs a hand edit of a file. `secrets setup` guides the operator through all steps on a terminal (an operator command). `secrets setup --print` shows the steps that remain and their commands, and changes nothing.
- **Identities.** An identity is a name for one age key file of this machine (`identities.toml`). One machine can have several identities, for example one for the store of the team and one for the store of a client. For each command, the CLI selects one key file: `--identity`, `SECRETS_IDENTITY`, `SOPS_AGE_KEY_FILE`, the identity of the project in `projects.toml`, the default identity, then `~/.config/sops/age/keys.txt`. You use the identity that the CLI selects.
- **The project index.** `projects.toml` lists the projects of this machine: paths, names, stores, and identities, never values. Each command that loads a `secrets.toml` records its project. `secrets project add <folder>` adds the projects under a folder. `secrets project list` shows the index.
- **The UI.** `secrets ui` is a local web page for the operator: the stores, the projects, and the values (hidden until a reveal). The operator can set values there. You never run it and never read values from it.

The full model is in [[dev-knw-nsec-model]]. The command reference, the exit statuses, and the known limits are in [[dev-knw-nsec-cli]]. The setup of a new team, a new machine, and a project is in [[dev-knw-nsec-setup]].

## The Rules in Short

The complete rules are in [[dev-knw-nsec-rules]]. Read them before your first `secrets` or `secret` command.

1. Never print, log, echo, or type a value. Not in output, a commit message, a Mesh note, a message, a test fixture, or a command.
2. Never write a value to a file. No `.env` file, no `secrets export … > file`. If a tool needs a file, stop and ask the operator.
3. Never pass a value as a process argument. A program reads a value from its process environment.
4. Never read or copy a key file (the key file of each identity, `$SOPS_AGE_KEY_FILE`, `~/.config/sops/age/keys.txt`, and `<secrets home>/keys/`). For the public key, run `secrets keygen --public`. Never decrypt a store file. Never edit `*.enc.yaml`, `.sops.yaml`, `.gitleaks.toml`, `recipients.toml`, `identities.toml`, or `projects.toml`.
5. Never run `secrets setup` (except `setup --print`), `set`, `edit`, `unset`, `keygen` (except `keygen --public` with no `--file`), `recipient add|rm`, `store init`, `store fix-config`, `store remove`, `push`, `login`, `logout`, `identity add|use|rm`, `project identity <name>`, `project identity --clear`, or `project forget`, unless the operator asks for that exact command. Never add `--yes` to a push.
6. Ask the operator for a missing value with the exact command: `secrets set <name> --env <env>`, or the page of `secrets ui`. Then continue with `secret <cmd>`. Never ask for a value in the chat.
7. You are a user. You have the access of your operating-system user. The rules prevent accidents. They are not a security boundary.
8. If a value leaks, stop, do not repeat it, and tell the operator the variable and the place. The operator replaces the value.
9. Never set `SECRETS_GITLEAKS=0`, and never remove, rename, or hide gitleaks on `PATH`. Before a store write that this shard allows, check for an inherited `SECRETS_GITLEAKS=0` and report it. When the secret scan stops a write (`SECRET_SCAN_FOUND`, `SECRET_SCAN_FAILED`), stop and report. See [[dev-knw-nsec-rules]] § Rule 10.
10. Change `secrets.toml`, `secrets.local.toml`, and the NUU Secrets lines of `.gitignore` only with the commands of the CLI (`init-project`, `bind`, `unbind`, `target set|rm`, `project gitignore`, `project set`), not by hand. Write a target table or change the store URL only with the ids or the URL that the operator gives.
11. Never run `secrets ui`, also when the operator asks. Never open its URL or login link, its page, or its API, and never read a value from it. The UI is for the operator. See [[dev-knw-nsec-rules]] § Rule 11.
12. Use the identity that the CLI selects. Do not add `--identity`, and do not set `SECRETS_IDENTITY` or `SOPS_AGE_KEY_FILE`, unless the operator names the identity. When a command or `doctor` reports `LINKED_MAPPING`, `MAPPING_CHANGED`, `OUTSIDE_PROJECT`, `DUPLICATE_PROJECT`, an unknown identity, `PROJECT_INDEX`, the link warning, or the warning about the identity of a parent project, stop and tell the operator the exact message and the remedy of the CLI. See [[dev-knw-nsec-rules]] § Rule 12.

## Situations

| Situation | Use |
|---|---|
| Run an app, a test, or a script with its secrets | [[dev-sk-nsec-run_app]] |
| A project has no `secrets.toml`, or a variable has no entry | [[dev-sk-nsec-setup_project]] |
| Help an operator set up NUU Secrets: a new team (a new store), a new machine, or the first project | [[dev-knw-nsec-setup]]; run `secrets setup --print` first |
| The shell cannot find `secrets` or `secret` | [[dev-knw-nsec-cli]] § Install: ask the operator to install the CLI |
| `secret` or `secrets` fails, or access is not correct | [[dev-sk-nsec-diagnose]] |
| A store write stops with the secret scan, or `doctor` warns about gitleaks | [[dev-sk-nsec-diagnose]] (step 9) and [[dev-knw-nsec-rules]] § Rule 10 |
| `doctor` warns `.gitleaks.toml differs from the text that the CLI generates` | [[dev-sk-nsec-diagnose]] (step 4): give the operator the exact `secrets store fix-config --store <name>` command; never run it |
| A value or a key is missing | [[dev-sk-nsec-request_value]] |
| A command or `doctor` reports an identity problem (`LINKED_MAPPING`, `MAPPING_CHANGED`, `OUTSIDE_PROJECT`, `DUPLICATE_PROJECT`, an unknown identity, a parent-identity warning) | Stop. [[dev-sk-nsec-diagnose]] (step 6) and [[dev-knw-nsec-rules]] § Rule 12 |
| The operator asks about identities, the project index, or `secrets ui` | [[dev-knw-nsec-setup]] § Identities, § The project index, § The UI; give the operator the commands |
| The task asks you to use `secrets ui`, or the operator sends a link `http://127.0.0.1:<port>/#t=…` | Do not run or open it. [[dev-knw-nsec-rules]] § Rule 11 |
| Prepare a push to Vercel, Convex, or Railway | [[dev-sk-nsec-plan_push]] |
| A question about the store, names, targets, or literals | [[dev-knw-nsec-model]] |
| A question about a command, an option, an exit status, or a message | [[dev-knw-nsec-cli]] |

## Lifecycle

The usual order of the work in one project:

```
setup_project → request_value → (operator: secrets set or secrets ui) → run_app
                      ↑                                        │
                      └──────────── diagnose ←── a failure ────┘
```

## Commands at a Glance

```sh
secret pnpm dev                          # run with dev values
secret --env preview node server.js      # run with preview values
secrets doctor                           # check everything offline; changes nothing
secrets doctor --online                  # after a plain doctor with the same options: refresh (fetch and reset), then check
secrets whoami                           # the id, the identity, the key fingerprint, the stores, the groups
secrets identity list                    # the identities, their public keys and stores, and the active selection
secrets project list                     # the project index: name, store, identity, state, path
secrets project identity                 # show the identity of this project; changes nothing
secrets project add <folder> --dry-run   # find each secrets.toml under the folder; without --dry-run, add to projects.toml
secrets store list                       # the store clones of this machine
secrets keygen --public                  # the public key of this machine; makes no key
secrets setup --print                    # the state of each setup step and the commands that remain; changes nothing
secrets init-project --name <project> --store <url>   # also adds .env* and secrets.local.toml to .gitignore
secrets vars --unmapped                  # the variable names that the code reads and no mapping has
secrets bind <VAR> <name> --to local
secrets unbind <VAR>                     # remove one entry
secrets target set vercel --project <id> # a target table, with the ids from the operator
secrets project gitignore                # add the missing .gitignore lines
secrets plan <target> --env <env>        # what a push sends: names and statuses; no platform call
```

Use the `secrets` command that is on `PATH`. Its build mode (prod or dev) changes the default secrets home. See [[dev-knw-nsec-model]] § The store.

When the shell cannot find `secrets`, do not install it yourself. Ask the operator to install the CLI: `npm install -g @nuucognition/secrets-cli@beta` (Node.js 24 or later). See [[dev-knw-nsec-cli]] § Install.
