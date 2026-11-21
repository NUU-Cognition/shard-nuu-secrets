---
description: "The complete agent rules for NUU Secrets: no value in output, files, or arguments; secrets pull and the block of .env.local (never read or change it, change a value in the store, report a managed name outside the block); no access to the key or the store files; mapping changes only through the commands; the operator-only commands (also secrets setup, the identity changes, and the changes of names, collections, expiry dates, refs, and vault entries); how to ask for a value; the secret scan of store commits; agents never use secrets ui; use the identity that the CLI selects and stop on an identity problem; moved and removed refs and expiry warnings; agents never read or write a vault text; what to do after a leak"
---

# Knowledge: Agent Rules

These rules apply to each task that touches NUU Secrets: a project with a `secrets.toml`, a `secrets pull`, a `.env` file, a key, or a store. Read all the rules before you run a `secrets` command. When a rule and a task do not agree, the rule wins: stop and tell the operator.

## Words

This shard uses one word for one thing.

| Word | Meaning |
|---|---|
| value | The plaintext of a secret of an environment, for example an API key, a token, or a password. |
| secret name | The stable name of a value in the store, for example `clerk/secret-key`. A secret name is not secret. |
| collection | The first segment of a secret name: `openai` in `openai/api-key`. A collection can have a record in `collections.toml` (a description, a source URL, a warning window). A name with one segment is in no collection. |
| expiry date | The first day on which the key of one secret name in one environment does not work, in `metadata.toml`. The operator sets it. The CLI warns when the date is near (`due`) or past (`expired`). |
| change log | `changes.jsonl` in the store: one line for each change of a write (a name added, changed, removed, or moved; a collection or expiry change; a vault change), with the time and the recipient id of the writer. It never holds a value or a vault text. |
| vault | The encrypted free-text entries of a store, in `vault/<group>/<name>.enc.yaml`. The vault is not part of the environments. |
| vault entry | One entry of the vault: an entry name (for example `aws/root-login`), a group (`dev` or `prod`), and an encrypted text. An entry name is not secret. |
| vault text | The plaintext of a vault entry, for example a password, a login, recovery codes, or a private key. Agents never read or write it (Rule 14). |
| moved ref, removed ref | A `secret://` ref of a project that has no value, and that the change log moved to another name (`moved`) or removed (`removed`). |
| block | The lines of the `.env.local` of a project between the start marker `# >>> nuu-secrets · store <revision> (<date>) · env <env> · sha256 <hash> · do not edit: …` and the end marker `# <<< nuu-secrets <<<`. `secrets pull` owns these lines. They hold values. All other lines of `.env.local` belong to the person or to another tool (for example `npx convex dev` or Vercel). |
| pull | One write of the block from the store by `secrets pull`. The direction is one way: store → `.env.local`. |
| managed name | A variable that `secrets.toml` or `secrets.local.toml` gives to the target `local`. Its line belongs in the block. |
| variable | An environment variable of a process, for example `CLERK_SECRET_KEY`. A variable name is not secret. |
| environment | The store environment that `--env` selects: `dev`, `preview`, `prod`, or `deploy`. |
| store | The Git repository of the team that holds the encrypted values (SOPS + age). |
| store clone | The copy of the store on this machine, at `<secrets home>/stores/<store name>/`. The CLI owns it as a cache. |
| key file | An age private key file of this machine. For each command, the CLI selects one key file: the key file of an identity, `$SOPS_AGE_KEY_FILE`, or `~/.config/sops/age/keys.txt` ([[dev-knw-nsec-model]] § Identities and the key selection). |
| identity | A name for one key file of this machine, in `identities.toml` of the secrets home. The file holds names and paths, never a key. One machine can have several identities. An identity is not a security boundary between people. |
| project index | `projects.toml` in the secrets home: the projects of this machine, with the path of each `secrets.toml`, the project name, the store name, and the project identity. It holds no value. |
| the UI | `secrets ui`: the local web page for the operator. It shows the stores, the projects, the collections, the change log, the vault, and the values and vault texts (hidden until a reveal). Agents never use it (Rule 11). |
| store write | A command that changes the store with one Git commit and one push: `secrets store init`, `set`, `unset`, `edit`, `mv`, `collection create\|edit\|rename\|rm`, `expiry set\|clear`, `vault edit\|set\|mv\|rm`, `recipient add\|rm`, and `store fix-config`, and each write of the page of `secrets ui`. Each store write is an operator command (Rule 5). |
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

Do not run a command that shows the process environment: `env`, `printenv`, `set`, `export -p`, `echo "$VAR"`, `cat /proc/<pid>/environ`, or a script that prints `process.env`. Do not run a command that shows a line of `.env.local` (Rule 15).

To check that a variable has a value, use the safe signals. None of them shows a value:

- The `pull` line on stderr gives the count of variables and the count of `optional` entries with no value. A required `ref` with no value stops the pull, so a successful pull proves that each required variable of the target `local` is in the block.
- `secrets pull --check` says `current` or `stale`, with the names only.
- `secrets doctor` names each required secret name that has no value, and checks the block.

An app can show a value in its output: debug output, a configuration dump, or an error with a connection URL. Do not start an app with debug or dump options. If a value shows, follow [If a value leaks](#if-a-value-leaks).

## Rule 2 — Never write a value to a file

- Do not write `.env`, `.env.local`, `.env.*.local`, or another file that holds values. The one exception: `secrets pull` writes the block of `.env.local` (Rule 15). You run the command; you never write a line of the file yourself.
- Do not send the output of `secrets export` to a file (`>`, `>>`, or `tee`).
- Do not copy a value into a test fixture, a configuration file, a script, or a temporary file.
- Do not open or show the content of a `.env*` file. This also applies to `.env.local` after a pull, and to an example file such as `.env.example`, also when Git tracks it. Any of these files can hold a value, and a file can be a symbolic link to another file. `secrets doctor` names the managed variables in the `.env*` files, and it shows names only. Get the variable names of an app from its source code ([[dev-sk-nsec-setup_project]], step 4).
- If a tool can read its configuration only from a file, stop. Tell the operator the tool and the file. The operator decides.
- Never run `vercel env pull`. It writes the values of a Vercel environment to a file.
- `secrets pull` writes only the block of `.env.local`, only for `dev` and `preview`, and keeps each other line byte for byte. Never make another file with values, for example with a redirect of `secrets export` (`--format dotenv > .env.local`): it would replace the lines of other tools, and it has no block.
- `npx convex dev` writes its own lines into the same `.env.local` (`CONVEX_DEPLOYMENT`, `NEXT_PUBLIC_CONVEX_URL`), outside the block. The Convex tool owns these names, and a push never sends them to Convex. Keep them out of the target `local` of `secrets.toml` unless the operator decides otherwise: a managed name outside the block stops each pull (Rule 15). Do not open or change these lines.

## Rule 3 — Never pass a value as a process argument

Other processes can read the arguments of a process. A program must read a value from its process environment or from stdin.

- Do not write a command in which a shell expands a value into the arguments of a program. For example, a shell that loads `.env.local` and runs `some-cli "$API_TOKEN"` gives the value to `some-cli` as an argument. Use a program that reads the variable itself. Never load `.env.local` into a shell (`source .env.local`, `set -a`).
- `secrets bind --value` is only for a literal that is not secret. Never give a value to `--value`. `bind` refuses some values that look like secrets, but its check does not find all of them.
- Do not give a value to a platform CLI (Vercel, Convex, Railway) as an argument.

## Rule 4 — Never touch the key or the store files

- Do not open, print, copy, move, or change a key file. This applies to each key file of each identity (the `KEY FILE` column of `secrets identity list`) and to each file in `<secrets home>/keys/`. To check the key, run `secrets doctor` (the key selection, the path, the mode, and the fingerprint), `secrets whoami` (the identity, the fingerprint, and the path), `secrets identity list` (the public key of each identity), or `secrets keygen --public` (the full public key of the selected key file; it makes no key). The fingerprint is the last 8 characters of the public key.
- Do not read or print the variable `SOPS_AGE_KEY`: it can hold a private key. Do not run the command of `SOPS_AGE_KEY_CMD`.
- Do not decrypt a store file. Do not run `sops` on a store file. An error of `secrets pull` can say "Run sops yourself to see it". That sentence is for the operator, not for you.
- When a read stops with ``✘ The store file <file> of store <store> has SOPS metadata that the CLI does not decrypt. SOPS did not run, and nothing was read.`` (`UNSAFE_STORE`), or with the same message for a vault file (``✘ The vault file <path> of store <store> has SOPS metadata …``, `UNSAFE_VAULT`), stop. The file has a key of another system (for example `hc_vault`, `kms`, or `pgp`), an unknown metadata key, or no SOPS metadata, and SOPS could send a credential of this machine to that system. Do not open, edit, or decrypt the file, and do not set a variable of another key system. Give the operator the exact message ([[dev-sk-nsec-diagnose]], step 4).
- Do not edit `*.enc.yaml`, `.sops.yaml`, `.gitleaks.toml`, `recipients.toml`, `collections.toml`, `metadata.toml`, `changes.jsonl`, `stores.toml`, `identities.toml`, or `projects.toml` by hand. Do not open, decrypt, copy, or change a file under `vault/` of a store clone (Rule 14). Do not add an `identity` key to `secrets.local.toml`: a project file never selects an identity, and the CLI ignores the key with a warning (Rule 12).
- Change `secrets.toml`, `secrets.local.toml`, and the NUU Secrets lines of `.gitignore` with the commands of Rule 6 (`init-project`, `bind`, `unbind`, `target set|rm`, `project gitignore`, `project set`), or with `refs fix` after the approval of the operator (Rule 13), not by hand. These commands check the result and refuse a bad change. When a command cannot edit an unusual form of the file (`MAPPING_EDIT`), report it to the operator.
- Do not run Git commands in a store clone, and do not delete a store clone. Do not delete a `.removing-*` folder in the stores folder: report it to the operator. To refresh a store clone, first run plain `secrets doctor` for that store (Rule 6). Then run `secrets store sync <name>`, or `secrets doctor --online` with the same options.
- Do not move, link, or replace folders in the secrets home. Managed-clone checks refuse linked folders below the real secrets home. Store-file checks refuse non-regular files. Both refuse a file or a folder that they cannot check (`… cannot check …`, `… cannot be checked (<code>)`). Not every metadata command runs every check. Report each refusal to the operator. Do not change modes to pass a check.

## Rule 5 — Operator-only commands

The operator enters values, manages keys, and owns the names, the collections, the expiry dates, and the vault. Do not run these commands. The one exception: the operator asks for that exact command now, or approves it (the approval of the operator). (A test of the CLI in a sandbox has its own rule: see [Tests of the CLI itself](#tests-of-the-cli-itself).)

| Command | What it does |
|---|---|
| `secrets setup` (without `--print`) | The guided setup on a terminal. It makes keys and stores and writes values ([[dev-knw-nsec-setup]]). `secrets setup --print` is not on this list (see Rule 6). |
| `secrets set` | Writes a value. The value comes from a hidden prompt or stdin. `--expires <date>` also sets its expiry date. |
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
| `secrets mv <from> <to>` | Renames a secret in each store file that holds it. Then the refs of the projects that use the old name are `moved`. See Rule 13. |
| `secrets collection create`, `edit`, `rename`, `rm` | Change a collection record. `rename` also renames each secret of the collection, as `mv` does. See Rule 13. |
| `secrets expiry set`, `secrets expiry clear` | Set or remove the expiry date of one secret in one environment. See Rule 13. |
| `secrets refs fix` (without `--dry-run`) | Rewrites each moved ref in `secrets.toml` and `secrets.local.toml`. `secrets refs fix --dry-run` is not on this list (see Rule 6). See Rule 13. |
| `secrets vault mv`, `secrets vault rm` | Rename a vault entry, move it to the other group, or remove it. See Rule 14. |
| `secrets pull --force` | Replaces a block of `.env.local` that a person or a program changed after the last pull. `secrets pull` without `--force` is not on this list (see Rule 6). See Rule 15. |

`secrets ui`, `secrets vault show`, `secrets vault edit`, and `secrets vault set` are not on this list: an agent never runs them, also when the operator asks (Rules 11 and 14).

"The operator asks" and "the approval of the operator" mean: the operator (or your dispatcher, for the operator) names this command, with its arguments, for this action in this session. A general task, for example "set up the secrets of the app" or "fix the refs", does not give this permission.

This exception never covers `--yes` on a push to production, unless the operator gives that exact command with `--yes`.

For `secrets set` and `secrets edit`, the operator types the values in their own terminal. Give the operator the command. Do not run it: you would have to type the value. Use [[dev-sk-nsec-request_value]].

## Rule 6 — Commands that you can run

When interrupted local work can matter, run plain `secrets doctor` before a refresh. Keep its lines about unpushed commits, changed files, and links. Check every store that the refresh will affect. Use the same options for plain and online doctor. Use `secrets store sync <name>` for a checked store. After operator work, run the plain check again before a refresh.

| Command | Use |
|---|---|
| `secrets pull` | Write the `local` values of the project (`dev` or `preview`) into the block of `.env.local`. Then run the normal command of the program. Without `--force`. See Rule 15 and [[dev-sk-nsec-run_app]]. |
| `secrets pull --check` | Show whether the block is current (`current` or `stale`, with names only). It writes nothing. Run it before each pull. |
| `secrets pull --all`, `secrets pull --check --all` | The same for each project of the project index in the current Git repository. A refusal of one project stops the pull before the first write. |
| `secrets doctor` | Check the tools, the key, the stores, the mapping, and the project. Without `--online`, it changes nothing. See [[dev-sk-nsec-diagnose]]. |
| `secrets doctor --online` | The same checks after a refresh: it fetches each checked clone and resets it to the remote. The reset discards the local changes of an interrupted command in the clone (a cache that the CLI owns). It never clones a missing store. Run it only after a plain `secrets doctor` with the same options. |
| `secrets keygen --public` | Print the full public key of the key file of this machine. It makes no key and never prints the private key. Use it only with no `--file`. |
| `secrets whoami` | Show the recipient id, the selected identity, the key fingerprint, and the stores of this machine. |
| `secrets identity list` | List the identities: the names, the public keys, the key file paths and their state, the default, the stores and groups of each identity, and the `active:` line (the key that the CLI selects here). No private key. Do not open the key files that it names (Rule 4). |
| `secrets project list` | List the project index: name, store, identity, the last load, the state (`ok`, `missing`, `linked`, `duplicate`), and the path. No values. |
| `secrets project add <path>` | Add the projects under a folder (or one `secrets.toml`) to the project index. It writes only `projects.toml`, as each command that loads a `secrets.toml`: a new entry has no identity, and a known entry keeps its identity. It changes no project file, and it reads no store and no value. Run it with `--dry-run` first. |
| `secrets project identity` (no name, no `--clear`) | Show the identity of the project of the current folder (or of `--mapping <path>`). It changes nothing. |
| `secrets store list` | List the store clones of this machine (no values). |
| `secrets store sync <name>` | Refresh one store clone that a plain `secrets doctor` checked first. Always name the store. It also shows the change-log entries since the last sync, the refs of the projects of this machine that they break, and the due and expired secrets. |
| `secrets log` | Show the change log of the store: names, environments, vault groups, dates, and the recipient id of each writer. No value, no vault text, and no key needed. |
| `secrets refs` | Show the state of each ref of the project (or with `--all`, of each project of the project index): `ok`, moved, removed, or not in the store. No key needed. See Rule 13. |
| `secrets refs fix --dry-run` | Show the plan of the ref fix. It changes nothing. Without `--dry-run`, `refs fix` needs the approval of the operator (Rule 5). |
| `secrets collection list` | List the collections: the record, the number of names in each environment, the warning window, and the next expiry date. No key needed. |
| `secrets expiry list` | List the expiry dates with their state (`ok`, `due`, `expired`). No key needed. |
| `secrets vault list` | List the vault entries: the name, the group, the last change, and the access of this machine. It decrypts nothing and shows no text. See Rule 14. |
| `secrets store clone` | Clone the store of a project (a network access). You seldom need it: `secrets pull` without `--offline` clones the store when necessary. |
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

- Use `secrets pull` and the normal command of the program when you can. It covers almost every case.
- Use `secrets export` only when a tool must read the values from stdin. Pipe the output directly into that tool, in the same command. That tool must not print what it reads.
- Never run `secrets export` without a pipe. Never pipe it into a program that shows its input, for example `cat`, `less`, `head`, `jq`, or `tee`.
- Do not use `eval "$(secrets export --format shell)"`. The values then stay in your shell. Use `secrets pull`.
- Never redirect `secrets export` into `.env.local` or into another file (Rule 2).
- Do not use `secrets export --to vercel`, `--to convex`, or `--to railway` to send values to a platform. Only the operator sends values to a platform, with `secrets push` (Rule 9).

## Rule 7 — Ask the operator for values and keys

When a value or a key is missing, ask the operator. Give the variable, the secret name, the environment, and the exact command for the operator. A missing ref with a `moved` or `removed` hint is not a missing value: follow Rule 13. A vault text is not a value of an environment: when a task needs a password, a login, or another text from the vault, follow Rule 14. The operator sets a value with `secrets set <name> --env <env>` in their own terminal, or in the page of `secrets ui` (Rule 11). Then you continue with `secrets pull`. Never ask the operator to paste a value into the chat, a file, or a message. Never guess a value, and never use a placeholder value that looks real. Use [[dev-sk-nsec-request_value]].

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
- **Other refusals of a store write.** `UNSAFE_VAULT` (`✘ The vault of store <store> has a path that the CLI does not accept.`), `UNSAFE_STORE` (`✘ A file of store <store> breaks the store policy. …`), and `UNSAFE_RESULT` (`✘ The result of a SOPS write breaks the store policy. …`) name a file label, a line number, and a field name, for example `the SOPS metadata field "version" does not have the form that SOPS writes`. Report these facts to the operator. Do not retry the write. Do not open the store file. For `UNSAFE_FILE`, `COMMIT_FAILED`, `WRITE_FAILED`, and `COMMIT_MISMATCH`, give the fixed text to the operator, and do not change the store clone ([[dev-knw-nsec-cli]] § The messages of a store write).
- **No gitleaks on `PATH`.** The warning is `secrets: gitleaks is not on PATH, so the secret scan of the store commit did not run. …`. The scan did not run. Check the command's exit status. If the command succeeded, the write was made without a scan. Report both facts. Ask the operator to install gitleaks.
- **What to copy.** Copy the fixed write-error text, safe rule labels, safe file labels, field names, and cleanup warnings. Do not copy file content or raw tool output. Some error lines show a path that the user chose or set. The output rules of the CLI do not cover the text of that path. Do not copy a line when its path or another field holds a value: name the kind of error.
- **Manual commits.** Change the managed store only through `secrets`. The CLI scan does not cover a manual commit in another checkout. Never make a Git commit in a store clone yourself (Rule 4).

In a headless session, a scan refusal and an inherited `SECRETS_GITLEAKS=0` are blockers. See [[dev-hinit-nsec]].

## Rule 11 — The UI (`secrets ui`) is for the operator

`secrets ui` starts a local web page on `127.0.0.1`. The page shows the stores, the projects, the collections, the change log, the vault, and the values of this machine. A reveal or a copy in the page gives a value or a vault text. Thus the UI is for a person at a terminal, never for an agent.

- **Never run `secrets ui`.** This applies to each option (`--no-open`, `--identity`, `--idle`), and also when the operator asks. With no terminal, the CLI refuses: `✘ secrets ui needs a terminal on stdin and stdout: the UI is for a person at a terminal. Nothing was started.` (`NO_TERMINAL`). Do not give it a terminal (a pseudo-terminal, `script`, `expect`, `tmux`, or a terminal tool of your harness).
- **Never read values from the UI.** This also applies to a vault text, and to the actions of the page (for example "Fix the refs" and the expiry form). Do not open the URL or the login link of the UI. Do not open the page in a browser or in a browser tool. Do not send a request to its server or its API (`curl`, `fetch`, or another HTTP client to `127.0.0.1:<port>`). Do not read its launch file: it holds the login token.
- **A login link is a credential.** The link has the form `http://127.0.0.1:<port>/#t=<token>`. When the operator sends you a link of this form, do not open it, and do not repeat the token. Tell the operator that the link is now in the session transcript. The token works one time, in the first 60 seconds. For a new link, the operator stops `secrets ui` and runs it again.
- **The reveal log is for the operator.** `<secrets home>/ui-reveal.log` holds one line for each reveal or copy: the time, the identity, and the store, the environment, and the name (for a literal: the project and the variable), never the value; for a vault entry: the group and the entry name, never the text. Do not change or delete it.
- **What you do.** You use `secrets pull` and the normal command to run a program with the values. You ask the operator to set a value in the UI or with `secrets set` ([[dev-sk-nsec-request_value]]). The writes of the page are operator writes: they use the write path of the CLI (the lock, the policy check, the secret scan, the commit, and the push). When the operator sends you an error of the page, read it as a message of the CLI ([[dev-sk-nsec-diagnose]]).

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
  | `UNKNOWN_IDENTITY` | `✘ There is no identity named "<name>"` (then ` (<where>)` and the end of the sentence) |
  | `PROJECT_INDEX` | `✘ The CLI cannot read the identity of project <name>: …` |
  | the parent warning | `secrets: warning: <path> has no identity of its own in projects.toml, so the CLI uses <identity or key file>. The secrets.toml above it in the same project, <path> (project <name>), has the identity <name>. …` |
  | the link warning | `secrets: warning: … so the identity <name> of its target <path> in projects.toml does not apply (a link can name the secrets.toml of another project). …` |

  The two warnings do not stop the command: the command ran with another key than the project identity, often with the default identity. Stop after the command, and report the warning before more work in that project. The other problems stop the command: nothing was read or written.
- **An identity is not a security boundary.** Each process of your operating-system user can read each key file and change `identities.toml` and `projects.toml`. The rules prevent accidents ([An agent is a user](#an-agent-is-a-user)).

In a headless session, an identity problem is a blocker. See [[dev-hinit-nsec]].

## Rule 13 — Moved refs, collections, and expiry dates

The change log records each change of a store write. The CLI uses it to explain a missing ref. The operator owns the secret names, the collections, and the expiry dates.

- **You can read them.** `secrets log`, `secrets refs`, `secrets refs fix --dry-run`, `secrets collection list`, `secrets expiry list`, and `secrets vault list` show names, environments, groups, dates, and recipient ids, never a value (Rule 6).
- **A change needs the approval of the operator.** `secrets mv`, `secrets collection create|edit|rename|rm`, `secrets expiry set|clear`, `secrets refs fix` (without `--dry-run`), and `secrets vault mv|rm` (Rule 5). Each of them except `refs fix` is a store write: check for an inherited `SECRETS_GITLEAKS=0` first (Rule 10).
- **A moved ref.** The error of a missing value, a `doctor` line, a `plan` line, or `secrets refs` says ``moved to secret://<new name> on <date> by <recipient id>; run `secrets refs fix` ``. The CLI never uses the value of the new name: the project file must change first. Run `secrets refs fix --dry-run` with the same `--mapping`, `--project`, or `--all`. Give the operator the plan, and ask for the approval of the fix. Change the files only after the approval ([[dev-sk-nsec-diagnose]], step 7). Do not edit the ref by hand, do not `bind` the new name yourself, and do not run the program from another folder or with another `--mapping` to pass the check.
- **A removed ref.** The hint says `removed on <date> by <recipient id>` (or `moved to …, then removed on …`). The store has no value for that name now. Ask the operator what to do: bind another name, remove the entry, or write a new value. Do not decide it yourself: do not `unbind` the entry, do not bind another name, and do not mark the entry `optional`.
- **A ref that is not in the store** with no change-log entry is a missing value (Rule 7).
- **A push.** `plan` and `push` stop on a moved ref, also on an `optional` ref. The refs change first (with the approval of the operator), then the plan.
- **An expiry warning.** `secrets pull`, `pull --check`, `export`, `doctor`, and `store sync` warn when the expiry date of a secret is near (`due`) or past (`expired`), for example ``secrets: warning: <VAR> (secret://<name>, <env>) expires on <date> (in <n> days). Rotate the key, then run `secrets expiry set <name> --env <env> --date <new date>`.`` The warning does not stop the command (a pull still writes the block). Tell the operator the variable, the secret name, the environment, and the date. Do not run `secrets expiry set` or `secrets expiry clear`, and do not change a date to stop a warning. The operator replaces the key at its provider, then writes the new value and its date (`secrets set <name> --env <env> --expires <date>`). An `expired` key can make the program fail with an authentication error: report the warning with the failure. Never read the value to check it.
- **A move changes refs.** `secrets mv` and `secrets collection rename` change secret names. After such a change, the refs of the projects that use the old names are `moved`.

In a headless session, a moved ref and a removed ref are blockers. An expiry warning is not a blocker: put it in your result. See [[dev-hinit-nsec]].

## Rule 14 — The vault is for the operator

The vault keeps free text that is not an environment variable: passwords, logins, recovery codes, and private keys. A vault text is as secret as a value. Rules 1, 2, and 3 apply to a vault text as to a value. A vault text never reaches your context.

- **Never read or write a vault text.** Never run `secrets vault show` (also with `--copy`), `secrets vault edit`, or `secrets vault set`. This applies also when the operator asks: the operator runs them in their own terminal. With no terminal, `vault show` and `vault edit` refuse (`NO_TERMINAL`) with the last hint `An agent does not run secrets vault show or secrets vault edit. It asks the operator for the text.` Do not give them a terminal (a pseudo-terminal, `script`, `expect`, `tmux`, or a terminal tool of your harness).
- **Never read the clipboard** (`pbpaste`, `wl-paste`, `xclip -o`, `xsel -o`, or a clipboard tool of your harness). After `secrets vault show --copy` or a copy in `secrets ui`, the clipboard holds a vault text or a value for 30 seconds or more: the clear after 30 seconds happens only when the clipboard still holds that text, and only when a read tool exists.
- **Never touch a vault file.** Do not open, decrypt, copy, or change a file under `vault/` of a store clone (Rule 4). Do not write a file with a vault text, and do not send a text to the stdin of `secrets vault set`.
- **What you can run.** `secrets vault list` shows the entry names, the groups, the last change, and the access of this machine. It decrypts nothing and shows no text (Rule 6). `secrets vault mv` and `secrets vault rm` need the approval of the operator (Rule 5): each one is a store write (Rule 10).
- **A task needs a text from the vault.** For example a password for a login, recovery codes, or a private key. Ask the operator. Give the entry name (when `secrets vault list` shows it) and what the task needs. The operator does the step that needs the text, for example the login. Then you continue. Never ask the operator to paste the text into the chat, a file, a message, or a command. When the operator pastes it anyway, follow [If a value leaks](#if-a-value-leaks).
- **The vault is not part of the environments.** `secrets pull`, `export`, `plan`, and `push` never give a vault entry to a program or to `.env.local`, and no `secrets.toml` entry maps a vault entry. Never tell the operator to put a value of an environment into the vault, or a vault text into `secrets.toml` (also not as a literal). When a program needs a text of the vault in its process environment, tell the operator. The operator decides.
- **The names are public.** The change log, `secrets log`, and `secrets store sync` name the vault entries that changed (the name and the group, never the text).

In a headless session, a task that needs a vault text is a blocker. See [[dev-hinit-nsec]].

## Rule 15 — `secrets pull` and the block of `.env.local`

`secrets pull` writes the values of the target `local` into one block of the `.env.local` in the folder of the `secrets.toml`. It is the one command that writes values to the disk, and only for `dev` and `preview`. The block is as secret as the store values. The direction is one way: store → `.env.local`.

- **Never read the block.** Never read, print, copy, or search the values in the block of `.env.local`. Do not open, print, search, or compare the file at all (`cat`, `less`, `head`, `grep`, `diff`, `git diff --no-index`, an editor, or a read tool of your harness): each line outside the block can hold a value too. The names, the counts, and the states come from `secrets pull`, `secrets pull --check`, and `secrets doctor`.
- **Change a value in the store, never in `.env.local`.** Never edit, add, remove, or comment out a line of the block. When a value must change, the operator changes the store (`secrets set <name> --env <env>`, or the page of `secrets ui`). Then you run `secrets pull` again. A hand change of the block stops the next pull (`PULL_CHANGED_BLOCK`).
- **`--force` needs the approval of the operator.** `secrets pull --force` replaces a block that was changed after the last pull. Run it only when the operator approves it for that project.
- **A managed name outside the block.** When `secrets pull` stops with `✘ <path> sets names that secrets.toml gives to local, outside the block of NUU Secrets:` and the lines `  - line <n>: <VAR>` (`PULL_MANAGED_OUTSIDE`), report each line number and name to the operator. Do not delete, change, comment out, or move the line without the approval of the operator. Do not remove the target `local` of the name, and do not unbind it, to make the pull pass. The operator decides: remove the line, or remove the target `local` of the name in `secrets.toml` (for a name of another tool, for example `NEXT_PUBLIC_CONVEX_URL`, the usual decision is the second one). Never read the line to see its value.
- **Pull before you run.** Before you run an app, run `secrets pull --check`. When it says `stale`, run `secrets pull`. Then run the normal command of the program. Never load `.env.local` into your shell (Rule 3).
- **No production values on the disk.** `secrets pull` refuses `--env prod` and `--env deploy` (`PULL_ENV`). Do not try another way (`secrets export` with a redirect, a copy of a platform variable) to put a production value into a file.
- **Git must ignore `.env.local`.** A pull refuses a `.env.local` that Git does not ignore (`PULL_NOT_IGNORED`): run `secrets project gitignore`. A pull also refuses a `.env.local` that Git tracks (`PULL_TRACKED`): the history can already hold values. Tell the operator. Run `git rm --cached .env.local` only with the approval of the operator.
- **Links.** A pull never writes through a symbolic link (`PULL_LINK`). Do not remove or replace a link to pass the check. Report it.
- **Project files.** Do not add `secrets pull` to the scripts of `package.json` (for example `predev`), to a `Makefile`, or to CI files. CI has no key.

The steps are in [[dev-sk-nsec-run_app]]. In a headless session, a managed name outside the block, a changed block (a `--force` needs the operator), a broken block, a tracked `.env.local`, and a link are blockers. See [[dev-hinit-nsec]].

## Tests of the CLI itself

This shard is for the use of NUU Secrets in projects. A task that tests the `secrets` CLI itself (not an app) needs a sandbox:

- In each command that runs `secrets`, set `HOME`, `SECRETS_HOME`, `NUU_HOME`, `XDG_RUNTIME_DIR`, `XDG_CONFIG_HOME`, and `TMPDIR` to temporary folders. Set `SOPS_AGE_KEY_FILE` to a file path in the temporary root folder (for example `<temp root>/home/keys.txt`). Unset `SOPS_AGE_KEY` and `SOPS_AGE_KEY_CMD`: SOPS can use them, and the SOPS default key file under `XDG_CONFIG_HOME`, as other key sources. Unset `SECRETS_GITLEAKS`: an inherited `SECRETS_GITLEAKS=0` turns off the secret scan of the test store (Rule 10). Unset `SECRETS_IDENTITY`: it selects an identity before `SOPS_AGE_KEY_FILE`. The temporary `SECRETS_HOME` has its own `identities.toml` and `projects.toml`, so the test does not change the identities or the project index of the operator.
- Use test keys (`age-keygen`), a local bare Git repository as the store remote, and made-up test values.
- Never read or change the real key file or the real store clones.
- Delete the temporary folders at the end.

Only the prompt of such a task can allow the operator-only commands, and only inside the sandbox. Rule 14 applies also in a sandbox: the test suite of the CLI repository runs the vault commands with made-up texts. You do not run `secrets vault show|edit|set` in your shell.

## An agent is a user

You have the access of your operating-system user. You can read the key file. You can decrypt each store file of the groups of this machine, also production values when this machine is in the group `prod`. You can read the `.env.local` of each project after a pull, and `/proc/<pid>/environ` of each process of the user. The rules are a promise that you keep. They are not a lock.

- Do not test the rules.
- Do not read a value to prove a weakness.
- When you find a way for a value to leak, describe it to the operator without the value.

## If a value leaks

A leak is a value in your context, in output, in a file, in a commit, in a message, or in a log. Act at once:

1. Stop the current work. Do not run a command that can show the value again.
2. Do not repeat the value. Do not quote it in your report, also not a part of it.
3. Tell the operator now. Give the variable, the secret name, the environment, and where the value went: the command, the file path, the commit, the message, or the log. Say that the session transcript holds the value.
4. If you wrote the value to an untracked local file, delete that file and say so. Do not delete or change `.env.local`: tell the operator. The operator changes the store, then a pull replaces the block. If the value reached a commit, a push, a Mesh note, an Orbh message, or another shared place, do not try to remove it. The operator decides about a change of history.
5. The operator replaces the value at its provider (rotation) and writes the new value with `secrets set <name> --env <env>`. Until then, the old value is public.

In a headless session, report the leak as a blocker (a subagent to its dispatcher, a root session to the operator), then stop. See [[dev-hinit-nsec]].
