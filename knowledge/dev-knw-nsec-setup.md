---
description: "The setup of NUU Secrets for an operator: the install, Git access, the guided setup secrets setup and setup --print, a new store for a new team, a new machine for a store, a project, the accepted store URL forms, and which steps an agent can do"
---

# Knowledge: The Setup for an Operator

Use this file when an operator sets up NUU Secrets: a new team (a new store), a new machine for a store of the team, or a new project. You guide the operator. You give the commands. The operator runs the operator commands ([[dev-knw-nsec-rules]] § Rule 5). Each step of the setup is a command of the CLI. No step needs a hand edit of a file.

## Who does what

| Step | Who | Commands |
|---|---|---|
| Install the CLI and the tools | Operator | `npm install -g @nuucognition/secrets-cli@beta`; SOPS, age, Git, gitleaks |
| The guided setup | Operator | `secrets setup` (on a terminal) |
| Show the state of the setup | Agent or operator | `secrets setup --print` (changes nothing) |
| The key of this machine | Operator | `secrets keygen` |
| The recovery key and a new store | Operator | `secrets keygen --out <file>`, `secrets store init <url> --recovery <age-public-key>` |
| Add a machine to a store | Operator (a store admin) | `secrets recipient add "<id>" <age-public-key> --groups <groups>` |
| Clone a store | Agent or operator | `secrets store clone <url>` |
| The mapping of a project | Agent or operator | `secrets init-project`, `secrets vars`, `secrets bind`, `secrets unbind`, `secrets project gitignore` |
| The platform targets | Agent with the ids from the operator | `secrets target set`, `secrets target rm` |
| The store URL of a project | Agent with the URL from the operator | `secrets project set --store <url>` |
| The values | Operator | `secrets set <name> --env <env>` |
| Check | Agent or operator | `secrets doctor` |

## Before the setup

- **The CLI.** `npm install -g @nuucognition/secrets-cli@beta` gives the commands `secrets` and `secret`. It needs Node.js 24 or later. See [[dev-knw-nsec-cli]] § Install.
- **The tools.** SOPS 3.11 or later, `age` and `age-keygen`, and Git are necessary. gitleaks is optional, but without it the CLI does not scan store commits (Rule 10). `secrets doctor` and step 1 of `secrets setup` check each tool and give its install command.
- **Git access.** The CLI uses Git for the store repository. The machine needs Git access to that repository: an SSH key that the Git host accepts, or a Git credential helper for HTTPS. Never put a credential in a store URL. The CLI refuses an HTTPS URL with user information and an SSH URL with a password.
- **The repository of a new store.** The CLI does not make the repository. The operator makes an empty private repository on the Git host first (no README, no first commit).

## The guided setup: `secrets setup`

`secrets setup` is an operator command. It makes keys, makes stores, and writes values. It needs a terminal on stdin and stdout. Without one, it stops with `✘ secrets setup needs a terminal on stdin and stdout. Nothing was changed.` and `Next: secrets setup --print`. Never run `secrets setup` without `--print`. Tell the operator to run it in their own terminal.

It has 8 steps. Each step checks the state first and skips a step that is done. A second run continues where the first run stopped.

1. **Tools.** The tool lines of `doctor`. A missing required tool, or SOPS older than 3.11, stops the setup.
2. **Key.** The key file of this machine, or a new key (as `secrets keygen`).
3. **Store.** The store that `secrets.toml` names, or one of: a new store, join a store of the team, or a store that is already on this machine.
4. **Project.** `secrets.toml` in the current folder, and the `.gitignore` lines.
5. **Targets.** Vercel, Convex, Railway, or none, with their fields.
6. **Variables.** The names that the code reads (as `secrets vars`). For each name: a store name, a literal (a hidden prompt; the secret check runs first), or skip. Then the targets and `optional`.
7. **Values.** For each store name with no dev value: enter the dev or the prod value now (a hidden prompt, as `secrets set`), or skip.
8. **Check.** Plain `secrets doctor`, then the next commands.

Facts:

- **Exit status.** The setup ends with the exit status of the final `doctor`. Thus the setup exits 1 while a check fails, for example while a required prod value is missing. That is not a failure of the setup. Read the `→` lines of `doctor`.
- **Stops.** The setup stops (exit 1) on a safety refusal (for example `UNSAFE_CLONE`), on a store URL that holds a credential or that the CLI cannot classify (`BAD_STORE_URL`), on a store mismatch (`STORE_MISMATCH`: the values step writes only to the store that `secrets.toml` names), and when this machine is not a recipient yet (`NOT_RECIPIENT`). Ctrl-C at a prompt stops with exit 130. Each write is atomic: a stop leaves no half-written file.
- **No value in output.** The setup never shows a value or a private key. A refused literal is never shown.

## `secrets setup --print` (agents can run it)

`secrets setup --print` changes nothing, takes no lock, and makes no network call. It reads the cached clones as plain `doctor` does. When this machine is in group `dev` of a cached clone, it decrypts `dev.enc.yaml` in memory to find the store names with no dev value (names only). It always exits 0.

Output:

```text
secrets setup --print: the state of each step. Nothing was changed.
1. tools      done  sops 3.13.3, age 1.3.2, age-keygen 1.3.2, git 2.56.0, gitleaks …
2. key        todo  no age key file at <path>
     $ secrets keygen
3. store      todo  <path>/secrets.toml names the store <url>; this machine has no clone of it
       send the id and the public key to a store admin; compare the fingerprint through a second channel; the admin runs:
     $ secrets recipient add "<id>" <age-public-key> --groups dev
…
setup: 3 done, 4 todo. Run `secrets setup` on a terminal for the guided setup.
```

- Each step line is `<n>. <title>  <state>  <text>`. The titles are `tools`, `key`, `store`, `project`, `targets`, `variables`, `values`, `check`. The states are `done`, `todo`, `skip`, `wait`.
- A line `     $ <command>` is a command for that step. A line with more indent and no `$` is a note.
- Read the commands of each `todo` step. Run the commands that this shard lets you run (Rule 6). Give the operator the operator commands (Rule 5).

## A new store (a new team)

Do this one time for each team. The operator runs each command. With `secrets setup`, step 3 "a new store" does the same steps.

1. The key of this machine: `secrets keygen`. It prints the id, the public key, and the fingerprint.
2. The paper recovery key:
   ```sh
   secrets keygen --out ~/team-secrets-recovery-key.txt
   ```
   - The file is a private key. Put it in a folder that is not in a Git work tree. `secrets setup` refuses a path below a `.git` entry.
   - Print the file. Keep the paper in a safe place. Then delete the file. A deleted file can stay on an SSD or on a copy-on-write file system: the paper is the recovery key.
   - `secrets setup` writes the file with mode 600, waits until the operator types `yes` (the paper copy exists), and then deletes the file. On `stop`, the file stays, and a second run uses the public key of the same file. It refuses an existing file with a mode other than 600 (`chmod 600 <path>`).
3. An empty private repository on the Git host, for example `git@github.com:my-team/team-secrets.git`.
4. The store:
   ```sh
   secrets store init git@github.com:my-team/team-secrets.git --recovery age1…
   ```
   This machine and the recovery key become recipients of all groups (`dev`, `prod`, `deploy`).
5. The first values: `secrets set <name> --env <env>` (a hidden prompt). See [[dev-sk-nsec-request_value]].

## A new machine for a store

The new machine needs its own key. A key never moves between machines.

1. On the new machine (operator): `secrets keygen`. Or run `secrets setup` and pick "join a store of your team": it shows the id, the public key, the fingerprint, and the exact admin command.
2. The operator sends the id and the public key to a store admin. The admin compares the fingerprint (the last 8 characters of the public key) through a second channel, for example a call.
3. The admin, on a machine that is a recipient:
   ```sh
   secrets recipient add "<id>" <age-public-key> --groups dev
   ```
   Add `prod` or `deploy` only when the operator decides.
4. On the new machine: `secrets store clone <url>`, or let `secret <cmd>` clone it from `secrets.toml`. `secrets setup` checks again until the admin has added the machine, or stops with `NOT_RECIPIENT` and the hint to run `secrets setup` again later.
5. Check: `secrets whoami` (the groups) and `secrets doctor`.

When you help on the new machine, use the key request of [[dev-sk-nsec-request_value]] (step 4).

## A project

You can do these steps (Rule 6). Use [[dev-sk-nsec-setup_project]]. In short:

```sh
secrets init-project --name <project> --store <url>   # also adds .env* and secrets.local.toml to .gitignore
secrets vars --unmapped                               # the names that the code reads and no mapping has
secrets bind <VAR> <name> --to local                  # one for each variable
secrets target set vercel --project <name-or-id>      # only with the ids from the operator
secrets doctor --env dev
```

The operator then writes the missing values with `secrets set`.

## Store URL forms

The CLI accepts only these store URL forms:

- The scp form: `git@github.com:my-team/team-secrets.git`, or an SSH host alias: `github-work:my-team/team-secrets.git`.
- `ssh://`, `git+ssh://`, `ssh+git://`, `git://`, `http://`, `https://`, with a host name or a bracketed IPv6 address and an optional port: `ssh://git@host.example:2222/~/team-secrets.git`, `https://host.example/my-team/team-secrets.git`.
- `file:///<path>` and `file://<host>/<path>`.
- A path on this machine, for example `/srv/git/team-secrets.git` (a local bare repository).

Each other text is not a usual form: for example a `?`, a `#`, or a `\` anywhere, more than one `@`, a scheme that is not in the list, a `_` in the host of a URL (an scp alias can have `_`), or an `@` in the path. The CLI never shows such a text. It shows `<a store URL that the CLI cannot show>`. `init-project --store`, `project set --store`, and the URL questions of `secrets setup` refuse it (`BAD_STORE_URL`). Then ask the operator for the URL in a usual form.
