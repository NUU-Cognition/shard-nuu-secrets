---
description: "Diagnose a NUU Secrets failure from what the CLI reports: run secrets doctor, act on each warn and fail (what an agent fixes, what goes to the operator), and read secrets whoami, store list, and store sync"
---

> [!important] THIS FILE IS AN INSTRUCTION. WHEN REFERENCED IT IS MEANT TO BE TAKEN AS AN ACTION.

Run `flint shard start nsec` if you haven't already.

# Skill: Diagnose

Find what NUU Secrets reports, fix the parts that an agent may fix, and send the other parts to the operator. Report what the CLI says. Do not guess a cause that the output does not show. No step of this skill shows a value.

# Input

- The failure: the command, its exit status, and its `✘` message
- The project folder (it has `secrets.toml`, or a parent folder has it), or no project
- (Optional) The environments that the task needs

# Actions

1. **Read the rules.** You read [[dev-knw-nsec-rules]]. Never run `sops`, never open the key file, and never open a store file to diagnose.
2. **Read the cached state first, then refresh.** First run `secrets doctor` once with no `--online` (step 3). It changes nothing, and it shows unpushed commits, changed files, and links in the clone. A refresh discards or hides them, so keep these lines for the report. Then, when the network is available, run:
   ```sh
   secrets store sync
   ```
   It fetches each clone and resets it to the remote. It also shows each change of `recipients.toml` since the last sync, with the ids, groups, fingerprints, and commit authors. Read the output with step 8. Then run `secrets doctor` again.
3. **Run the checks.** In the project folder, run:
   ```sh
   secrets doctor
   ```
   - Without `--online`, `doctor` reads the cached clones, makes no network call, and changes nothing. The line `cached revision <revision> (<date>): doctor reads the clone offline, so it can be older than the remote` says how old the clone is.
   - `secrets doctor --online` first fetches each checked clone and resets it to the remote, then checks. The reset discards the local changes and commits of an interrupted command (the clone is a cache that the CLI owns). It never clones a missing store. It does not show the changes of `recipients.toml`: use `secrets store sync` for them.
   - Outside the project folder, add `--mapping <path>` or `--project <name>`.
   - Add `--env <env>` (one or more times) to check only some environments. The default is `dev`, `preview`, and `prod`.
   - The exit status is 1 when a check fails. The last line counts the results. An exit status 0 can still have `warn` and `skipped` lines.
4. **Act on each `fail`, `warn`, and important `skipped` line.** A check can have a remedy line (`→ …`). Use this table to decide who acts. "Operator" means: do not do it yourself; put it in the report of step 9.

   | Section | Message (start) | Who acts | Action |
   |---|---|---|---|
   | Tools | `<tool> is not on PATH`, `<tool> does not start`, `sops <version> is too old` | Operator | Give the remedy line (the install command). |
   | Key | `fail  no age key file at <path>`, `fail  the age key file <path> is not a regular file`, `fail  the age key file <path> cannot be read` | Operator | Use the key request of [[dev-sk-nsec-request_value]] (step 4). The operator makes a key (`secrets keygen`) or sets `SOPS_AGE_KEY_FILE` to the file of an existing key. Do not make a key. |
   | Key | `warn  no age key file at <path>; SOPS can decrypt with <source>` | — | SOPS has another key source (`SOPS_AGE_KEY`, `SOPS_AGE_KEY_CMD`, or the SOPS default key file). Nothing to fix for SOPS. Report it once. Do not read `SOPS_AGE_KEY`, and do not run the command of `SOPS_AGE_KEY_CMD`. |
   | Key | `skipped  the public key of SOPS_AGE_KEY_CMD (the CLI does not run the command)` | — | The key of this machine is unknown to `doctor`. See the `unknown` rows. |
   | Key | `key file <path> has mode <mode>: other users can read it` | Operator | Give the remedy (`chmod 600 <path>`). Do not change the key file. |
   | Key | another `fail` | Operator | Give the message. |
   | Store | `this machine is not a recipient of the store` | Operator | Look at the Key section first. When it has a `fail`, use that row. When the Key section shows a public key, the key file exists: use the key request of [[dev-sk-nsec-request_value]] (step 4). You get the public key with `secrets keygen --public`. The operator does not make a new key. An admin adds it with `secrets recipient add`. |
   | Store | `warn  this machine as a recipient: unknown (SOPS_AGE_KEY_CMD gives the key, and doctor does not run it)` | — | This is not proof that this machine is not a recipient. A `secret <cmd>` run shows if the key decrypts. Report it. Do not request a new key. |
   | Mapping | `skipped  <env>: the access is unknown (doctor does not run SOPS_AGE_KEY_CMD)` | — | The same as the row above. |
   | Store | `ok  cached revision …: doctor reads the clone offline, so it can be older than the remote` | Agent | Normal. For a current result, run `secrets doctor --online` (or `secrets store sync`, then `doctor`). |
   | Store | `fail  --online: cannot refresh the clone: …` and `warn  cached revision …: the refresh failed, …` | Agent, then operator | Give the exact line. The cause can be the network, the remote, the local clone, or the access. Retry once. Then report. |
   | Store | `fail  <file> is a symbolic link (<path>)`, `fail  <file> is committed as a symbolic link (mode 120000) (<path>)`, `fail  <file> is committed as a folder (<path>)`, `fail  <file> is not a regular file (<path>)` | Operator | A store file is a link or a special file. `doctor` then skips the other checks of that store. Give the lines. The repair is an operator task (in a checkout of the store: `git rm` the link, commit, push; then `secrets store fix-config`). Do not change the clone. |
   | Store | `✘ The stores folder <path> is a symbolic link …` | Operator | Give the message. Do not move or link folders in the secrets home. |
   | Store | `.sops.yaml does not match recipients.toml: …` | Operator | Give the line. `doctor` shows that the two files do not agree. It does not show which file changed. Do not run `secrets store fix-config`. |
   | Store | `warn  .gitleaks.toml is missing`, `warn  .gitleaks.toml differs from the text that the CLI generates` | Operator | Give the remedy line (`secrets store fix-config --store <name>`). It is an operator command. |
   | Store | a problem of a store file, or `group <group> has no recipient` | Operator | Give the message. |
   | Store | `the store has no recovery key` | Operator | Give the remedy line. |
   | Store | `a stale lock of process <pid>` | Agent | The next CLI command removes it. Run your command again. Do not delete the lock file. |
   | Store | `<n> unpushed commit(s) … from an interrupted command` | Operator | A write of the operator stopped. The operator runs that command again. `doctor --online` and `store sync` discard such commits: report the line before you run them. |
   | Store | `the clone has no commit`, `The clone of store <name> is missing` | Operator | Give the message and the secrets home. Do not delete or edit a clone. |
   | Store | `the store of the mapping is not cloned on this machine` | Agent | When the task allows network access, run the app with `secret <cmd>` (without `--offline`) or run `secrets store clone <url>`. `doctor --online` does not clone. First check that the secrets home is the intended one ([[dev-knw-nsec-model]] § The store). |
   | Stores | `there is no store on this machine` | Operator | With no mapping, the operator decides which store to clone or create. |
   | Mapping | `<env>: <VAR> → secret://<name> is not in the store` | Operator | Request the value with [[dev-sk-nsec-request_value]]. For `preview`, a dev value is enough. |
   | Mapping | `<env>: no access (this machine is not in group <group>)` (skipped) | — | Normal when this machine does not need that environment. Report it only when the task needs that environment. |
   | Mapping | `<env>: sops decrypt failed …` | Operator | The key file is usable, but SOPS did not decrypt the file with it. Give the line. See the Key and Store rows. |
   | Mapping | `<VAR> goes to convex: a NEXT_PUBLIC_ variable is for the browser build` | Agent | Remove only `convex` from the targets of that entry. Keep the rest. See step 5. |
   | Mapping | `<VAR>: the literal in <file> looks like a secret` | Operator | A value can be in a committed file. Do not repeat it. Follow [[dev-knw-nsec-rules]] § If a value leaks. |
   | Mapping | `<path> is not valid:` with a list of problems | Agent | Fix each named entry of `secrets.toml` or `secrets.local.toml`. The problems name fields, not values. |
   | Project hygiene | `.gitignore does not cover …` | Agent | Add `.env*` and `secrets.local.toml` to `.gitignore`. |
   | Project hygiene | `<file> has names that secrets.toml manages: <VAR>, …` | Operator | A `.env*` file has duplicates. Do not open the file. The operator removes the names. |
   | Project hygiene | `secrets.local.toml is tracked by Git` | Agent and operator | Run `git rm --cached secrets.local.toml` and cover it in `.gitignore`. Tell the operator: the Git history keeps the old file. |
   | Machine | `full-disk encryption: …` | Operator | Report it once. It does not stop the work. |

   A command that refreshes first (`secret`, `export`, `store sync`, `doctor --online`) resets the work tree of the clone. It can remove a local link and then succeed. Report each `doctor` line about a link also when a later command succeeds.

5. **Remove `convex` from a `NEXT_PUBLIC_*` entry.** Change only the targets. Keep the `ref` or the literal, and keep `optional`.
   - Look at the line of the variable in `[env]` of `secrets.toml`. It has no value.
   - When `convex` is the only target, do not change the entry. Report it to the operator: the operator decides the targets.
   - A `ref` entry: run `bind` with the same secret name and the other targets. Add `--optional` when the entry has `optional = true`. Without `--optional`, `bind` makes the entry required.
     ```sh
     secrets bind <VAR> <name> --to <the other targets>
     secrets bind <VAR> <name> --to <the other targets> --optional
     ```
   - A literal entry (`value = "…"`): run `bind` with the same literal and the other targets. A literal is not secret.
     ```sh
     secrets bind <VAR> --value <the same literal> --to <the other targets>
     ```
   - You can also edit the `to` list of that one line by hand and remove only `"convex"`.
   - Run `secrets doctor` again. The `warn` for this variable must be gone.

6. **Read the identity.** Run `secrets whoami` when the problem is access.
   - A `key` row with ``no key (run `secrets keygen`)`` means that the CLI cannot read a public key from the key file. The file can be missing, not readable, or not valid. Run `secrets doctor` and use its Key line.
   - A `store` row `<name>  <id> in groups <groups>` gives the groups of this machine. A machine in the group `dev` only cannot open `prod` or `deploy`.
   - A `store` row `<name>  not a recipient` means that the store does not have the key of this machine.
   - A `store` row ``<name>  cannot read the clone (run `secrets doctor`)`` means that a clone file cannot be read. Run `secrets doctor`.
   - The `Account` part is optional. "Not logged in" and ``Run `secrets login` to authenticate.`` are not a problem. No command needs a login, and `login` is an operator command.
   - For the full public key of this machine, run `secrets keygen --public`. It makes no key.
7. **List the store clones.** Run `secrets store list`.
   - Each row is `NAME`, `REVISION`, `URL`.
   - `missing` means that the clone folder is gone. `invalid clone folder` and `invalid entry` mean a damaged clone or a damaged `stores.toml`. These go to the operator. Do not edit `stores.toml`.
   - `secrets: there is no store on this machine` means that no store is cloned in this secrets home.
   - A folder `.removing-<name>-<random>` in the stores folder is the rest of a failed `store remove`. Report it to the operator. Do not delete it.
   - An unwanted clone (for example in the wrong secrets home) goes to the operator: only the operator runs `secrets store remove <name>`.
8. **Read the output of `secrets store sync`.**
   - `<store>  up to date @ <revision>` or `<store>  <old> → <new>`: normal.
   - `secrets: store <name> had <n> unpushed commit(s) from an interrupted command. They are removed; …` or `secrets: store <name> had changed files from an interrupted command. They are removed.`: the refresh reset the clone. Report it. Any refresh can print these lines (`secret`, `export`, `store sync`, `doctor --online`).
   - `The history of the remote changed (a forced push).`: tell the operator.
   - `recipients.toml changed in <n> commit(s) since the last sync:` with `+`, `-`, or `~` lines: a recipient was added, removed, or changed. Copy these lines (ids, groups, fingerprints, and authors are public) into your report for the operator.
   - `secrets: store <name>: <message>` and `<n> of <m> store(s) did not sync.`: the sync of that store failed. Give the exact line. The cause can be the network, the remote, the local clone, a link below the secrets home, or the configuration. Retry once. Then report.
9. **Report.** Write one report with:
   - What you fixed, and the `doctor` result after your fixes.
   - Each item for the operator: the message, the exact command for the operator, and why.
   - Never a value. Names, ids, public keys, fingerprints, paths, and revisions are public.

# Output

- What the CLI reports about the failure (observations, not guesses)
- The agent fixes done, with a `secrets doctor` result after them
- A report for the operator with each remaining item and its exact command
