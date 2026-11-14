---
description: "The setup of NUU Secrets for an operator: the install, Git access, the guided setup secrets setup and setup --print, a new store for a new team, a new machine for a store, a project, identities (the least access as the default), the project index, the local web page secrets ui (operator only), the accepted store URL forms, and which steps an agent can do"
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
| The values | Operator | `secrets set <name> --env <env>`, or the page of `secrets ui` |
| The identities of this machine | Operator | `secrets identity add\|use\|rm`, `secrets project identity <name>` (or `--clear`) |
| Show the identities and the project index | Agent or operator | `secrets identity list`, `secrets project list`, `secrets project identity` |
| Add the projects of a folder to the project index | Agent or operator | `secrets project add <folder>` (first with `--dry-run`), or "Add projects from a folder" in the page of `secrets ui` (operator) |
| Remove a project from the project index | Operator | `secrets project forget <name\|path>` |
| The local web page | Operator only (never an agent) | `secrets ui` |
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

The operator then writes the missing values with `secrets set`, or in the page of `secrets ui`.

## Identities

An identity is a name for one age key file of this machine ([[dev-knw-nsec-model]] § Identities and the key selection). Use identities when one machine needs more than one key: for example `work` for the store of the team and `client` for the store of a client. With no `identities.toml`, the CLI works as before: `SOPS_AGE_KEY_FILE`, else the default key file. Each command of this section is an operator command, except `identity list` and `project identity` with no name and no `--clear` ([[dev-knw-nsec-rules]] § Rule 12).

```sh
secrets identity add work --key ~/.config/sops/age/keys.txt   # an existing key file
secrets identity add client --key ~/.config/sops/age/client.txt
secrets identity add test                 # a new key in <secrets home>/keys/test.txt (mode 600)
secrets identity use work                 # the default identity (or: identity add … --default)
secrets identity list [--json]            # the public key, the key file and its state, the default, the stores and groups, the active line
secrets identity rm test                  # the key file stays: the CLI never deletes a key
secrets project identity client           # the identity of the project of the current folder, on this machine
secrets project identity                  # shows it
secrets project identity --clear          # removes it: the default applies again
secret --identity work pnpm dev           # the identity for one command
```

- **Make the identity with the least access the default.** Then a project identity gives more access only in the projects where you set it. In the other direction, each folder where the project identity does not apply gets the default: a new clone, a project with no identity, and a new `secrets.toml` in a subfolder of a project (for example from a `git pull`). For the last case, the CLI prints one warning, but the command does not stop.
- **A new key of `identity add <name>`** is a new recipient. The CLI prints the admin command (`secrets recipient add "<id>" <age-public-key> --groups dev`). The admin compares the fingerprint through a second channel, as for a new machine ([A new machine for a store](#a-new-machine-for-a-store)).
- **The project identity is a fact of this machine.** It is in `projects.toml`, never in a project file. `secrets project identity <name>` refuses a `secrets.toml` with a symbolic link in its path, a file outside the project of the current folder, and a file that changed during the command. The name must be in `identities.toml`.
- **The variables.** `SECRETS_IDENTITY=<name>` selects an identity for each command of a shell. `SOPS_AGE_KEY_FILE` selects a key file, not an identity, and wins over the project identity and the default. `secrets project identity` and `secrets identity use` print a note when one of them is set in the shell.
- **The checks.** `secrets doctor` shows `key selection: …` (the step that selected the key). `secrets whoami` shows the identity.
- **Limits.** An identity is not a boundary between people: each program of one operating-system user can read each key file of that user. For a boundary, use another operating-system user or another machine. A key file with a passphrase (`age -p`) is not supported: `identity add --key` refuses it. Run `secret <cmd>` from the real folder of the project, not through a folder link of a cloned repository.

When a command stops on an identity problem (`LINKED_MAPPING`, `MAPPING_CHANGED`, `OUTSIDE_PROJECT`, `DUPLICATE_PROJECT`, `UNKNOWN_IDENTITY`), the error lines give the remedy: for example `secrets project identity <name> --mapping <path>` (or `--clear`), `secrets project forget <path>`, `--identity <name>` for one command, or a path with no link. The operator selects the remedy.

## The project index

`projects.toml` in the secrets home lists the projects of this machine ([[dev-knw-nsec-model]] § The project index). Each command that loads a `secrets.toml` records its project. `secrets project add <folder>` adds the projects that no command loaded yet, for example the projects of a monorepo. Nobody edits the file by hand.

```sh
secrets project list [--json]             # NAME, STORE, IDENTITY, SEEN, STATE, PATH
secrets project add ~/code --dry-run      # find each secrets.toml under the folder; write nothing
secrets project add ~/code                # add the projects; writes only projects.toml
secrets project forget my-app             # a project name
secrets project forget ./apps/web         # a path: a secrets.toml or its folder (a word with "/", or that starts with "." or "~")
```

- The states are `ok`, `missing` (the path does not exist), `linked` (the path has a symbolic link: its identity does not apply), and `duplicate` (another entry is for the same file). stderr adds the remedy for each kind.
- `secrets doctor` has a section `Projects (projects.toml)`. It warns for each missing path (`→ when the project is gone, run secrets project forget <path>`) and for each duplicate entry (`→ run secrets project identity <name> --mapping <path> (or --clear): it makes the entries one entry`). It removes nothing.
- `project add` takes a folder or one `secrets.toml`. It follows no symbolic link and does not enter hidden folders, `node_modules`, `dist`, `build`, and other build folders. It stops at 12 folders deep, 50000 folders, 15 seconds, or 1000 files, and adds the files that it found. A new entry has no identity, and a known entry keeps its identity. It prints `added`, `updated`, `known`, or `skipped <reason>` for each file ([[dev-knw-nsec-cli]] § `secrets project add`). An agent can run it.
- `forget` removes all entries of one file. The path of a link removes only the entry of the link, never the entry of its target. When a word is a project name and also a relative path of another entry, `forget` refuses and removes nothing. An agent does not run `forget`: an entry can hold the project identity.

## The UI: `secrets ui`

`secrets ui` starts a local web page for the operator. The page shows the stores, the projects of this machine, and the names of each project and each environment. A value stays hidden until the operator reveals it. Agents never run `secrets ui` and never read values from it ([[dev-knw-nsec-rules]] § Rule 11). You can tell the operator about it, for example to set the missing values.

```sh
secrets ui [--identity <name>] [--no-open] [--idle <minutes>]
```

- **Start.** The operator runs `secrets ui` in a terminal. With no terminal on stdin and stdout, it refuses and starts nothing (`NO_TERMINAL`). The server listens on `127.0.0.1` only, on a random port. The terminal shows one URL: `http://127.0.0.1:<port>/#t=<token>`. Use this `127.0.0.1` URL, not `localhost`: the server accepts no other host name.
- **The login link.** The token works one time, in the first 60 seconds. The CLI opens the browser with a launch file (mode 600), so the token is in no process argument. With `--no-open`, the operator opens the URL. For a new link, the operator stops `secrets ui` and runs it again. The link is a credential: never send it in a chat or a message.
- **The projects.** The side bar of the page shows the projects of the project index. When the side bar has no projects ("This machine has no projects yet"), the operator uses "Add projects from a folder" (also in the side bar and in the ⌘K palette): give a folder in the home folder (`~` works), click "Find projects", choose the projects, and click "Add". It does the same as `secrets project add <folder>` (an agent can run that command for the operator). It writes only `projects.toml`.
- **The identity choice.** After the login, the page asks for an identity of `identities.toml` (`--identity <name>` selects it in advance). The page then reads and writes with the key file of that identity only: no `SOPS_AGE_KEY_FILE`, no `SECRETS_IDENTITY`, and no project identity applies. When `identities.toml` no longer names that key file for the identity, the page locks. With no identity on this machine, the page shows its first setup: it makes the first identity (the default) with a new key or an existing key file.
- **The lock.** The page locks after 15 minutes with no request (`--idle <minutes>`, 1 to 1440), and when the operator clicks Lock. To unlock, the operator chooses the identity again. The lock and Quit delete the Undo records of the server.
- **The hidden values.** A list never holds a value. The page shows 8 dots for a hidden value. A reveal gives one value and hides it again after 30 seconds, on blur, and on a change of the selection. Copy works without a reveal, and the page clears the clipboard after 30 seconds when it can. The page compares values ("same as dev") with no reveal. Before each reveal or copy, the server writes one line in `<secrets home>/ui-reveal.log`: the time, the identity, the store, the environment, the name, and show or copy, never the value.
- **Writes.** Each write of the page uses the write path of the CLI: the lock, the policy check, the secret scan, the commit, and the push. A write to `prod` or `deploy` needs the typed environment name. An import makes one store commit for each environment. The page changes `secrets.toml`, `secrets.local.toml`, and `.gitignore` with the code of the setup commands; it does not commit the project repository. The terminal shows one line for each write and each sync, with names only.
- **A part of a write.** A write that changes the store and `secrets.toml` (an import with bind, an added variable, the removal of a variable, an Undo) can stop after one part. Then the page shows `Only a part of the change was saved.` (`PARTIAL_WRITE`) and offers "Undo the saved part" for 60 seconds. The terminal line names the store commits that stay and the file. The store commits stay pushed. To remove a variable, the page changes `secrets.toml` first and unsets the store values after it.
- **A flagged store name.** The page cannot unset a store name that the secret detector flags, also not with "Remove the variable". The operator uses `secrets unset <name> --env <env>` in a terminal.
- **Quit.** Ctrl-C in the terminal, Quit in the page, and a closed terminal stop the server.

## Store URL forms

The CLI accepts only these store URL forms:

- The scp form: `git@github.com:my-team/team-secrets.git`, or an SSH host alias: `github-work:my-team/team-secrets.git`.
- `ssh://`, `git+ssh://`, `ssh+git://`, `git://`, `http://`, `https://`, with a host name or a bracketed IPv6 address and an optional port: `ssh://git@host.example:2222/~/team-secrets.git`, `https://host.example/my-team/team-secrets.git`.
- `file:///<path>` and `file://<host>/<path>`.
- A path on this machine, for example `/srv/git/team-secrets.git` (a local bare repository).

Each other text is not a usual form: for example a `?`, a `#`, or a `\` anywhere, more than one `@`, a scheme that is not in the list, a `_` in the host of a URL (an scp alias can have `_`), or an `@` in the path. The CLI never shows such a text. It shows `<a store URL that the CLI cannot show>`. `init-project --store`, `project set --store`, and the URL questions of `secrets setup` refuse it (`BAD_STORE_URL`). Then ask the operator for the URL in a usual form.
