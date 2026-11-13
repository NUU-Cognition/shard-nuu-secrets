---
description: "The complete agent rules for NUU Secrets: no value in output, files, or arguments; no access to the key or the store files; mapping changes only through the commands; the operator-only commands (also secrets setup and the identity changes); how to ask for a value; the secret scan of store commits; agents never use secrets ui; use the identity that the CLI selects and stop on an identity problem; what to do after a leak"
---

# Knowledge: Agent Rules

These rules apply to each task that touches NUU Secrets: a project with a `secrets.toml`, a `secret <cmd>` run, a `.env` file, a key, or a store. Read all the rules before you run a `secrets` or `secret` command. When a rule and a task do not agree, the rule wins: stop and tell the operator.

## Words

This shard uses one word for one thing.

| Word | Meaning |
|---|---|
| value | The plaintext of a secret, for example an API key, a token, or a password. |
| secret name | The stable name of a value in the store, for example `clerk/secret-key`. A secret name is not secret. |
| variable | An environment variable of a process, for example `CLERK_SECRET_KEY`. A variable name is not secret. |
| environment | The store environment that `--env` selects: `dev`, `preview`, `prod`, or `deploy`. |
| store | The Git repository of the team that holds the encrypted values (SOPS + age). |
| store clone | The copy of the store on this machine, at `<secrets home>/stores/<store name>/`. The CLI owns it as a cache. |
| key file | An age private key file of this machine. For each command, the CLI selects one key file: the key file of an identity, `$SOPS_AGE_KEY_FILE`, or `~/.config/sops/age/keys.txt` ([[dev-knw-nsec-model]] § Identities and the key selection). |
| identity | A name for one key file of this machine, in `identities.toml` of the secrets home. The file holds names and paths, never a key. One machine can have several identities. An identity is not a security boundary between people. |
| project index | `projects.toml` in the secrets home: the projects of this machine, with the path of each `secrets.toml`, the project name, the store name, and the project identity. It holds no value. |
| the UI | `secrets ui`: the local web page for the operator. It shows the stores, the projects, and the values (hidden until a reveal). Agents never use it (Rule 11). |
| store write | A command that changes the store with one Git commit and one push: `secrets store init`, `set`, `unset`, `edit`, `recipient add|rm`, and `store fix-config`. Each store write is an operator command (Rule 5). |
| secret scan | The gitleaks scan that the CLI runs on the bytes of each store commit before the commit (Rule 10). |
| operator | The person who owns this machine and its store access. Only the operator enters values and manages keys. |

## Why the rules exist

You run as the operating-system user of the operator. Thus you have the access of that user. You can read the key file. You can decrypt each store file of the groups of this machine. You can read the process environment of each process of the user. NUU Secrets does not stop you. It protects the values at rest and in Git. It does not protect them from a process of the same user. The operator decided this: **an agent is a user.**

These rules prevent accidents. They are not a security boundary. Nothing else stops you, so you must keep them.

A value that you see goes into your context. Your context goes into the session transcript on disk and to the model provider. Thus a value in your context is a leak. The main rule: **a value never reaches your context, and a value never reaches a place that people or tools read.**

## Rule 1 — Never show a value

Never print, log, echo, or type a value. This applies to each of these places:

- A reply, a summary, a turn result, an Orbh message, an update, or a return.
- A commit message, a code comment, a source file, a test fixture, or a test snapshot.
- A Mesh note, a task, a report, or a notepad.
- A log line that you add to an app.
- A command that you type. A value in a command goes into the shell history and into the transcript.

Do not run a command that shows the process environment: `env`, `printenv`, `set`, `export -p`, `echo "$VAR"`, `cat /proc/<pid>/environ`, or a script that prints `process.env`. This rule also applies under `secret`.

To check that a variable has a value, test it and do not print it:

```sh
secret sh -c 'test -n "$CLERK_SECRET_KEY" && echo "CLERK_SECRET_KEY is set"'
```

Use single quotes. Then the inner shell reads the variable, and your shell does not expand it.

Use the safe signals first. The `secret:` line on stderr gives the count of variables. `secrets doctor` names each required secret name that has no value. Neither shows a value.

An app can show a value in its output: debug output, a configuration dump, or an error with a connection URL. Do not start an app with debug or dump options under `secret`. If a value shows, follow [If a value leaks](#if-a-value-leaks).

## Rule 2 — Never write a value to a file

- Do not write `.env`, `.env.local`, `.env.*.local`, or another file that holds values.
- Do not send the output of `secrets export` to a file (`>`, `>>`, or `tee`).
- Do not copy a value into a test fixture, a configuration file, a script, or a temporary file.
- Do not open or show the content of a `.env*` file. This also applies to an example file such as `.env.example`, also when Git tracks it. Any of these files can hold a value, and a file can be a symbolic link to another file. `secrets doctor` names the managed variables in the `.env*` files, and it shows names only. Get the variable names of an app from its source code ([[dev-sk-nsec-setup_project]], step 4).
- If a tool can read its configuration only from a file, stop. Tell the operator the tool and the file. The operator decides.
- Never run `vercel env pull`. It writes the values of a Vercel environment to a file.

## Rule 3 — Never pass a value as a process argument

Other processes can read the arguments of a process. A program must read a value from its process environment or from stdin.

- Do not write a command in which a shell expands a value into the arguments of a program. For example, `secret sh -c 'some-cli "$API_TOKEN"'` gives the value to `some-cli` as an argument. Use a program that reads the variable itself.
- `secrets bind --value` is only for a literal that is not secret. Never give a value to `--value`. `bind` refuses some values that look like secrets, but its check does not find all of them.
- Do not give a value to a platform CLI (Vercel, Convex, Railway) as an argument.

## Rule 4 — Never touch the key or the store files

- Do not open, print, copy, move, or change a key file. This applies to each key file of each identity (the `KEY FILE` column of `secrets identity list`) and to each file in `<secrets home>/keys/`. To check the key, run `secrets doctor` (the key selection, the path, the mode, and the fingerprint), `secrets whoami` (the identity, the fingerprint, and the path), `secrets identity list` (the public key of each identity), or `secrets keygen --public` (the full public key of the selected key file; it makes no key). The fingerprint is the last 8 characters of the public key.
- Do not read or print the variable `SOPS_AGE_KEY`: it can hold a private key. Do not run the command of `SOPS_AGE_KEY_CMD`.
- Do not decrypt a store file. Do not run `sops` on a store file. An error of `secret` can say "Run sops yourself to see it". That sentence is for the operator, not for you.
- Do not edit `*.enc.yaml`, `.sops.yaml`, `.gitleaks.toml`, `recipients.toml`, `stores.toml`, `identities.toml`, or `projects.toml` by hand. Do not add an `identity` key to `secrets.local.toml`: a project file never selects an identity, and the CLI ignores the key with a warning (Rule 12).
- Change `secrets.toml`, `secrets.local.toml`, and the NUU Secrets lines of `.gitignore` with the commands of Rule 6 (`init-project`, `bind`, `unbind`, `target set|rm`, `project gitignore`, `project set`), not by hand. These commands check the result and refuse a bad change. When a command cannot edit an unusual form of the file (`MAPPING_EDIT`), report it to the operator.
- Do not run Git commands in a store clone, and do not delete a store clone. Do not delete a `.removing-*` folder in the stores folder: report it to the operator. To refresh a store clone, first run plain `secrets doctor` for that store (Rule 6). Then run `secrets store sync <name>`, or `secrets doctor --online` with the same options.
- Do not move, link, or replace folders in the secrets home. Managed-clone checks refuse linked folders below the real secrets home. Store-file checks refuse non-regular files. Both refuse a file or a folder that they cannot check (`… cannot check …`, `… cannot be checked (<code>)`). Not every metadata command runs every check. Report each refusal to the operator. Do not change modes to pass a check.

## Rule 5 — Operator-only commands

The operator enters values and manages keys. Do not run these commands. The one exception: the operator asks you to run that exact command now. (A test of the CLI in a sandbox has its own rule: see [Tests of the CLI itself](#tests-of-the-cli-itself).)

| Command | What it does |
|---|---|
| `secrets setup` (without `--print`) | The guided setup on a terminal. It makes keys and stores and writes values ([[dev-knw-nsec-setup]]). `secrets setup --print` is not on this list (see Rule 6). |
| `secrets set` | Writes a value. The value comes from a hidden prompt or stdin. |
| `secrets edit` | Shows the values of one store file in an editor. For preview, it shows only the preview file. |
| `secrets unset` | Removes a value. |
| `secrets keygen`, `secrets keygen --out <file>`, `secrets keygen --public --file <path>` | Make an age private key, or read another key file. `secrets keygen --public` with no `--file` is not on this list (see Rule 6). |
| `secrets recipient add`, `secrets recipient rm` | Change who can open the store. |
| `secrets store init` | Creates a store. |
| `secrets store fix-config` | Writes the generated `.sops.yaml` and `.gitleaks.toml` again: one commit and one push. After cycle H, each existing store needs it once ([[dev-knw-nsec-cli]] § The config of stores). Report the `doctor` warning and the exact command to the operator. |
| `secrets store remove` | Removes a store clone and its entry in `stores.toml` (local state). The operator runs it from a folder outside the clone. |
| `secrets login`, `secrets logout` | Change the NUU account of the operator. |
| `secrets push` (every environment: `dev`, `preview`, `prod`) | Sends values to a platform (Vercel, Convex, Railway). It can remove variables, and on Convex the change takes effect at once. See Rule 9. |
| `secrets identity add`, `secrets identity use`, `secrets identity rm` | Add an identity (also a new key), change the default identity, or remove an identity. They change the key file that a command uses. See Rule 12. |
| `secrets project identity <name>`, `secrets project identity --clear` | Set or remove the identity of a project on this machine (in `projects.toml`). `secrets project identity` with no name and no `--clear` only shows it (Rule 6). See Rule 12. |
| `secrets project forget <name\|path>` | Removes the entries of a project from the project index. An entry can hold the project identity: without it, the default identity applies, and the default can have more access. See Rule 12. |

`secrets ui` is not on this list: an agent never runs it, also when the operator asks (Rule 11).

"The operator asks" means: the operator (or your dispatcher, for the operator) names this command for this action in this session. A general task, for example "set up the secrets of the app", does not give this permission.

This exception never covers `--yes` on a push to production, unless the operator gives that exact command with `--yes`.

For `secrets set` and `secrets edit`, the operator types the values in their own terminal. Give the operator the command. Do not run it: you would have to type the value. Use [[dev-sk-nsec-request_value]].

## Rule 6 — Commands that you can run

When interrupted local work can matter, run plain `secrets doctor` before a refresh. Keep its lines about unpushed commits, changed files, and links. Check every store that the refresh will affect. Use the same options for plain and online doctor. Use `secrets store sync <name>` for a checked store. After operator work, run the plain check again before a refresh.

| Command | Use |
|---|---|
| `secret <cmd>`, `secrets run` | Run a program with the values in its process environment. See [[dev-sk-nsec-run_app]]. |
| `secrets doctor` | Check the tools, the key, the stores, the mapping, and the project. Without `--online`, it changes nothing. See [[dev-sk-nsec-diagnose]]. |
| `secrets doctor --online` | The same checks after a refresh: it fetches each checked clone and resets it to the remote. The reset discards the local changes of an interrupted command in the clone (a cache that the CLI owns). It never clones a missing store. Run it only after a plain `secrets doctor` with the same options. |
| `secrets keygen --public` | Print the full public key of the key file of this machine. It makes no key and never prints the private key. Use it only with no `--file`. |
| `secrets whoami` | Show the recipient id, the selected identity, the key fingerprint, and the stores of this machine. |
| `secrets identity list` | List the identities: the names, the public keys, the key file paths and their state, the default, the stores and groups of each identity, and the `active:` line (the key that the CLI selects here). No private key. Do not open the key files that it names (Rule 4). |
| `secrets project list` | List the project index: name, store, identity, the last load, the state (`ok`, `missing`, `linked`, `duplicate`), and the path. No values. |
| `secrets project identity` (no name, no `--clear`) | Show the identity of the project of the current folder (or of `--mapping <path>`). It changes nothing. |
| `secrets store list` | List the store clones of this machine (no values). |
| `secrets store sync <name>` | Refresh one store clone that a plain `secrets doctor` checked first. Always name the store. |
| `secrets store clone` | Clone the store of a project (a network access). You seldom need it: `secret` without `--offline` clones the store when necessary. |
| `secrets setup --print` | Show the state of each setup step and the commands that remain. It changes nothing. See [[dev-knw-nsec-setup]]. |
| `secrets vars` | List the variable names that the code reads, and their mapping state. Names and paths only. |
| `secrets init-project`, `secrets bind`, `secrets unbind` | Write `secrets.toml` (or, with `--local`, `secrets.local.toml`): names and targets, no values. `init-project` also adds the `.gitignore` lines. See [[dev-sk-nsec-setup_project]]. |
| `secrets project gitignore` | Add the missing lines `.env*` and `secrets.local.toml` to `.gitignore`. |
| `secrets target set`, `secrets target rm` | Write or remove a `[targets.<target>]` table. Only with the ids that the operator or the task gives (Rule 8). |
| `secrets project set` | Change the store URL or the name in `[project]`. Change the store only with the URL that the operator gives. |
| `secrets plan <target> --env <env>` | Show what `secrets push` sends to a platform: names and statuses only. It calls no platform. See Rule 9 and [[dev-sk-nsec-plan_push]]. |
| `secrets export` | Only under the conditions below. |
| `--help`, `--version` | Show the help or the version. |

### Conditions for `secrets export`

`secrets export` writes values to stdout. If you run it in your shell, the values go into your context.

- Use `secret <cmd>` when you can. It covers almost every case.
- Use `secrets export` only when a tool must read the values from stdin. Pipe the output directly into that tool, in the same command. That tool must not print what it reads.
- Never run `secrets export` without a pipe. Never pipe it into a program that shows its input, for example `cat`, `less`, `head`, `jq`, or `tee`.
- Do not use `eval "$(secrets export --format shell)"`. The values then stay in your shell. Use `secret <cmd>`.
- Do not use `secrets export --to vercel`, `--to convex`, or `--to railway` to send values to a platform. Only the operator sends values to a platform, with `secrets push` (Rule 9).

## Rule 7 — Ask the operator for values and keys

When a value or a key is missing, ask the operator. Give the variable, the secret name, the environment, and the exact command for the operator. The operator sets a value with `secrets set <name> --env <env>` in their own terminal, or in the page of `secrets ui` (Rule 11). Then you continue with `secret <cmd>`. Never ask the operator to paste a value into the chat, a file, or a message. Never guess a value, and never use a placeholder value that looks real. Use [[dev-sk-nsec-request_value]].

## Rule 8 — Platform targets need the operator

A platform target (`vercel`, `convex`, `railway`) in `secrets.toml` tells `secrets push` what to send to a platform. Do not add a `[targets.<target>]` table, and do not add a platform target to the `to` list of an entry, unless the operator or the task names that target. Write the table with `secrets target set`, with the ids that the operator or the task gives (a Vercel project, a Convex deployment short name, a Railway project, service, and environments). Never guess an id.

## Rule 9 — Delivery to a platform

`secrets plan` shows what a push sends. `secrets push` sends it. A push sends values out of this machine, so only the operator runs it.

- **`secrets push` is an operator command for every environment** (`dev`, `preview`, `prod`). Do not run it. Give the operator the exact `plan` and `push` commands ([[dev-sk-nsec-plan_push]]).
- **Never add `--yes`.** `--yes` is the approval of the operator for a push to production. The exception of Rule 5 never covers `--yes`, unless the operator gives that exact command with `--yes`.
- **`secrets plan` is an agent command.** It prints names and statuses, never a value, and it calls no platform. But it decrypts the store values and the platform credentials in memory: it needs a key in group `deploy`. Before a plan, follow the refresh rule of Rule 6: when interrupted local work can matter, run plain `secrets doctor` with the same options first, or use `secrets plan --offline`.
- **`no access` is expected** for a key that is not in group `deploy`. It is not a permission to add access. Do not ask for group `deploy` unless the operator or the task says so.
- **A plan is not proof of a push.** `plan` cannot see the state of the platform: the Vercel variables of other environments, the Railway reference variables, or an identifier collision. A successful plan does not prove that the push passes.
- **Credentials.** Never write a platform credential. Give the operator the exact command: `secrets set vercel/token --env deploy`, `secrets set vercel/team-id --env deploy`, `secrets set convex/<deployment>/<env> --env deploy`, or `secrets set railway/token --env deploy`.
- **After an `uncertain` write, never run the push again.** Show the operator the confirmed writes and the uncertain writes from stderr. The CLI says: `Check each uncertain variable on the platform before you run the push again.`
- **A short store value inside a platform id.** A push stops when an id from the platform holds a store value or a credential, of any length. The operator decides. A value that is not secret can become a literal in `secrets.toml`. Never move a secret value into `secrets.toml`.
- **No deployment.** Do not deploy after a push, and do not run a platform CLI with a value. Give the operator the `next` lines of the push output.
- **Fixed error texts.** A push shows only fixed error texts and the HTTP status. Do not try to get the message of the platform from another source.

## Rule 10 — The secret scan of store commits

Before a store commit, the CLI scans the exact bytes of the change unless gitleaks is missing or `SECRETS_GITLEAKS=0`. A store write that changes the store makes a commit. A write with no change makes no commit and runs no scan. Each store write is an operator command (Rule 5). You run a store write only when the operator gives you that exact command. You also read the output of a store write that the operator sends you. The facts and the messages of the scan are in [[dev-knw-nsec-cli]] § The secret scan of store commits.

- **Never turn the scan off.** Never set `SECRETS_GITLEAKS=0`: not exported, not in a shell profile, not as a prefix of a command, and not in a script. Never remove, rename, or hide gitleaks on `PATH` to pass a write. Only the operator decides that a finding is a false positive. Only the operator runs one command with the prefix `SECRETS_GITLEAKS=0`.
- **Check for an inherited switch before a write.** Before you run a store write that this shard allows (an exact write command that the operator gives you, or a write in a sandbox test), check the environment of that command for `SECRETS_GITLEAKS=0`. Use the same shell as the write. Use one of these checks:
  ```sh
  test "${SECRETS_GITLEAKS-}" = 0 && echo "SECRETS_GITLEAKS=0 is set"
  secrets doctor     # the Tools line: SECRETS_GITLEAKS=0: the CLI does not scan store commits
  ```
  If the switch is set, do not continue silently. Do not run the write. Report it to the operator, and ask the operator to unset it.
- **`SECRET_SCAN_FOUND`.** Stop and report `SECRET_SCAN_FOUND`. Report only the safe count, rule labels, and file labels from the CLI. Preserve any cleanup warning. Do not open the matching line. Do not bypass or repeatedly retry the finding. A finding in an encrypted file can indicate plaintext metadata, modified tooling, or a false positive. Run `secrets doctor` and report the result. Only the operator decides whether a finding is a false positive.
  - The message starts with `✘ The secret scan (gitleaks) found <n> possible secret(s) in the change to store <store>.` The text output does not print the code.
  - The message has the hint ``Use `SECRETS_GITLEAKS=0` only as a prefix for one command after you confirm a false positive.`` This hint is for the operator. You never use it.
- **`SECRET_SCAN_FAILED`.** Run `secrets doctor` and report the fixed failure reason. Preserve any cleanup warning. Do not bypass a failed scan.
  - The message starts with `✘ The secret scan (gitleaks) of the change to store <store> did not complete: <reason>.`
- **Cleanup.** A scan refusal before staging writes no new Git object. Preserve any cleanup warning. A later failure can leave local objects. Do not assume that a reset removed them.
  - A message with `Nothing was pushed.` and the repair ``Run `secrets store remove <name>`, then `secrets store clone <url>`.``: report it to the operator. Run the repair only when the operator approves it. `store remove` is an operator command (Rule 5).
  - A cleanup line can name a folder that holds copies of the scanned bytes (`The CLI could not delete its private scan folder <folder>. Delete it.`). Do not open the files in that folder. Report the line to the operator. The operator deletes the folder.
- **Other refusals of a store write.** `UNSAFE_STORE` (`✘ A file of store <store> breaks the store policy. …`) and `UNSAFE_RESULT` (`✘ The result of a SOPS write breaks the store policy. …`) name a file label, a line number, and a field name, for example `the SOPS metadata field "version" does not have the form that SOPS writes`. Report these facts to the operator. Do not retry the write. Do not open the store file. For `UNSAFE_FILE`, `COMMIT_FAILED`, `WRITE_FAILED`, and `COMMIT_MISMATCH`, give the fixed text to the operator, and do not change the store clone ([[dev-knw-nsec-cli]] § The messages of a store write).
- **No gitleaks on `PATH`.** The warning is `secrets: gitleaks is not on PATH, so the secret scan of the store commit did not run. …`. The scan did not run. Check the command's exit status. If the command succeeded, the write was made without a scan. Report both facts. Ask the operator to install gitleaks.
- **What to copy.** Copy the fixed write-error text, safe rule labels, safe file labels, field names, and cleanup warnings. Do not copy file content or raw tool output. Some error lines show a path that the user chose or set. The output rules of the CLI do not cover the text of that path. Do not copy a line when its path or another field holds a value: name the kind of error.
- **Manual commits.** Change the managed store only through `secrets`. The CLI scan does not cover a manual commit in another checkout. Never make a Git commit in a store clone yourself (Rule 4).

In a headless session, a scan refusal and an inherited `SECRETS_GITLEAKS=0` are blockers. See [[dev-hinit-nsec]].

## Rule 11 — The UI (`secrets ui`) is for the operator

`secrets ui` starts a local web page on `127.0.0.1`. The page shows the stores, the projects, and the values of this machine. A reveal or a copy in the page gives a value. Thus the UI is for a person at a terminal, never for an agent.

- **Never run `secrets ui`.** This applies to each option (`--no-open`, `--identity`, `--idle`), and also when the operator asks. With no terminal, the CLI refuses: `✘ secrets ui needs a terminal on stdin and stdout: the UI is for a person at a terminal. Nothing was started.` (`NO_TERMINAL`). Do not give it a terminal (a pseudo-terminal, `script`, `expect`, `tmux`, or a terminal tool of your harness).
- **Never read values from the UI.** Do not open the URL or the login link of the UI. Do not open the page in a browser or in a browser tool. Do not send a request to its server or its API (`curl`, `fetch`, or another HTTP client to `127.0.0.1:<port>`). Do not read its launch file: it holds the login token.
- **A login link is a credential.** The link has the form `http://127.0.0.1:<port>/#t=<token>`. When the operator sends you a link of this form, do not open it, and do not repeat the token. Tell the operator that the link is now in the session transcript. The token works one time, in the first 60 seconds. For a new link, the operator stops `secrets ui` and runs it again.
- **The reveal log is for the operator.** `<secrets home>/ui-reveal.log` holds one line for each reveal or copy: the time, the identity, the store, the environment, and the name, never the value. Do not change or delete it.
- **What you do.** You use `secret <cmd>` to run a program with the values. You ask the operator to set a value in the UI or with `secrets set` ([[dev-sk-nsec-request_value]]). The writes of the page are operator writes: they use the write path of the CLI (the lock, the policy check, the secret scan, the commit, and the push). When the operator sends you an error of the page, read it as a message of the CLI ([[dev-sk-nsec-diagnose]]).

## Rule 12 — Identities: use the identity that the CLI selects

The CLI selects one key file for each command. The selection order is in [[dev-knw-nsec-model]] § Identities and the key selection. The identity decides which stores and groups a command opens. Thus a change of the identity can give a command more access, or less access.

- **Use the selection of the CLI.** Run your commands with the identity that the CLI selects. Do not add `--identity <name>`. Do not set, change, or unset `SECRETS_IDENTITY` or `SOPS_AGE_KEY_FILE`. The one exception: the operator (or the task, for the operator) names the identity for that command.
- **Do not change the identities.** Do not run `secrets identity add`, `secrets identity use`, `secrets identity rm`, `secrets project identity <name>`, `secrets project identity --clear`, or `secrets project forget`, unless the operator asks for that exact command (Rule 5). Do not edit `identities.toml` or `projects.toml` (Rule 4).
- **You can read them.** `secrets identity list`, `secrets project list`, `secrets project identity` (with no name and no `--clear`), `secrets whoami`, and `secrets doctor` (the line `key selection: …`) show the identities and the selection. They show no private key and no value (Rule 6).
- **Stop on an identity problem.** When a command or `secrets doctor` reports one of these problems, stop the work that needs the key. Tell the operator the exact message and the remedy lines (the hints under the `✘` line, or the `→` line of `doctor`). Do not apply a remedy yourself: no `--identity`, no `--clear`, no `forget`, no change of a link, and no run from another folder or with another `--mapping` to pass the check. The operator decides.

  | Problem (code) | The message starts with |
  |---|---|
  | `LINKED_MAPPING` | `✘ <path> is a symbolic link now, …` or `✘ <path> goes through the symbolic link <link> now, …`, then `and projects.toml has the identity <name> for that path. The CLI does not use the identity of a project through a link, …` |
  | `MAPPING_CHANGED` | `✘ The CLI cannot prove that <path> is the secrets.toml that it read: the file changed during the command. …` |
  | `OUTSIDE_PROJECT` | `✘ <path> is outside the project of the current folder (<boundary>). The identity <name> of that secrets.toml in projects.toml applies only in its own project (its folder, or its Git repository), …` |
  | `DUPLICATE_PROJECT` | `✘ projects.toml has <n> entries for one secrets.toml (<path>), with different identities:` |
  | `UNKNOWN_IDENTITY` | `✘ There is no identity named "<name>" (<where>).` |
  | `PROJECT_INDEX` | `✘ The CLI cannot read the identity of project <name>: …` |
  | the parent warning | `secrets: warning: <path> has no identity of its own in projects.toml, so the CLI uses <identity or key file>. The secrets.toml above it in the same project, <path> (project <name>), has the identity <name>. …` |
  | the link warning | `secrets: warning: … so the identity <name> of its target <path> in projects.toml does not apply (a link can name the secrets.toml of another project). …` |

  The two warnings do not stop the command: the command ran with another key than the project identity, often with the default identity. Stop after the command, and report the warning before more work in that project. The other problems stop the command: nothing was read or written.
- **An identity is not a security boundary.** Each process of your operating-system user can read each key file and change `identities.toml` and `projects.toml`. The rules prevent accidents ([An agent is a user](#an-agent-is-a-user)).

In a headless session, an identity problem is a blocker. See [[dev-hinit-nsec]].

## Tests of the CLI itself

This shard is for the use of NUU Secrets in projects. A task that tests the `secrets` CLI itself (not an app) needs a sandbox:

- In each command that runs `secrets` or `secret`, set `HOME`, `SECRETS_HOME`, `NUU_HOME`, `XDG_RUNTIME_DIR`, `XDG_CONFIG_HOME`, and `TMPDIR` to temporary folders. Set `SOPS_AGE_KEY_FILE` to a file path in the temporary root folder (for example `<temp root>/home/keys.txt`). Unset `SOPS_AGE_KEY` and `SOPS_AGE_KEY_CMD`: SOPS can use them, and the SOPS default key file under `XDG_CONFIG_HOME`, as other key sources. Unset `SECRETS_GITLEAKS`: an inherited `SECRETS_GITLEAKS=0` turns off the secret scan of the test store (Rule 10). Unset `SECRETS_IDENTITY`: it selects an identity before `SOPS_AGE_KEY_FILE`. The temporary `SECRETS_HOME` has its own `identities.toml` and `projects.toml`, so the test does not change the identities or the project index of the operator.
- Use test keys (`age-keygen`), a local bare Git repository as the store remote, and made-up test values.
- Never read or change the real key file or the real store clones.
- Delete the temporary folders at the end.

Only the prompt of such a task can allow the operator-only commands, and only inside the sandbox.

## An agent is a user

You have the access of your operating-system user. You can read the key file. You can decrypt each store file of the groups of this machine, also production values when this machine is in the group `prod`. You can read `/proc/<pid>/environ` of a process that `secret` started. The rules are a promise that you keep. They are not a lock.

- Do not test the rules.
- Do not read a value to prove a weakness.
- When you find a way for a value to leak, describe it to the operator without the value.

## If a value leaks

A leak is a value in your context, in output, in a file, in a commit, in a message, or in a log. Act at once:

1. Stop the current work. Do not run a command that can show the value again.
2. Do not repeat the value. Do not quote it in your report, also not a part of it.
3. Tell the operator now. Give the variable, the secret name, the environment, and where the value went: the command, the file path, the commit, the message, or the log. Say that the session transcript holds the value.
4. If you wrote the value to an untracked local file, delete that file and say so. If the value reached a commit, a push, a Mesh note, an Orbh message, or another shared place, do not try to remove it. The operator decides about a change of history.
5. The operator replaces the value at its provider (rotation) and writes the new value with `secrets set <name> --env <env>`. Until then, the old value is public.

In a headless session, report the leak as a blocker (a subagent to its dispatcher, a root session to the operator), then stop. See [[dev-hinit-nsec]].
