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
- **`secret <cmd>`.** It reads the store, decrypts in memory, and starts `<cmd>` with the values in its process environment. It writes no file with values. It can clone or refresh the encrypted store clone.
- **The operator.** Only the operator enters values (`secrets set`) and manages keys. You ask the operator for them.
- **No account.** A NUU account is optional. No command needs a login. NUU hosts no secrets.

The full model is in [[dev-knw-nsec-model]]. The command reference, the exit statuses, and the known limits are in [[dev-knw-nsec-cli]].

## The Rules in Short

The complete rules are in [[dev-knw-nsec-rules]]. Read them before your first `secrets` or `secret` command.

1. Never print, log, echo, or type a value. Not in output, a commit message, a Mesh note, a message, a test fixture, or a command.
2. Never write a value to a file. No `.env` file, no `secrets export … > file`. If a tool needs a file, stop and ask the operator.
3. Never pass a value as a process argument. A program reads a value from its process environment.
4. Never read or copy the key file (`$SOPS_AGE_KEY_FILE`, else `~/.config/sops/age/keys.txt`). Never decrypt a store file. Never edit `*.enc.yaml`, `.sops.yaml`, or `recipients.toml`.
5. Never run `secrets set`, `edit`, `unset`, `keygen`, `recipient add|rm`, `store init`, `store fix-config`, `login`, or `logout`, unless the operator asks for that exact command.
6. Ask the operator for a missing value with the exact command: `secrets set <name> --env <env>`. Never ask for a value in the chat.
7. You are a user. You have the access of your operating-system user. The rules prevent accidents. They are not a security boundary.
8. If a value leaks, stop, do not repeat it, and tell the operator the variable and the place. The operator replaces the value.

## Situations

| Situation | Use |
|---|---|
| Run an app, a test, or a script with its secrets | [[dev-sk-nsec-run_app]] |
| A project has no `secrets.toml`, or a variable has no entry | [[dev-sk-nsec-setup_project]] |
| `secret` or `secrets` fails, or access is not correct | [[dev-sk-nsec-diagnose]] |
| A value or a key is missing | [[dev-sk-nsec-request_value]] |
| A question about the store, names, targets, or literals | [[dev-knw-nsec-model]] |
| A question about a command, an option, an exit status, or a message | [[dev-knw-nsec-cli]] |

## Lifecycle

The usual order of the work in one project:

```
setup_project → request_value → (operator: secrets set) → run_app
                      ↑                                        │
                      └──────────── diagnose ←── a failure ────┘
```

## Commands at a Glance

```sh
secret pnpm dev                          # run with dev values
secret --env preview node server.js      # run with preview values
secrets doctor                           # check everything; changes nothing
secrets whoami                           # the id, the key fingerprint, the stores, the groups
secrets store list                       # the store clones of this machine
secrets init-project --name <project> --store <url>
secrets bind <VAR> <name> --to local
```

Use the `secrets` command that is on `PATH`. Its build mode (prod or dev) changes the default secrets home. See [[dev-knw-nsec-cli]] § Known limits.
