---
description: "Diagnose a NUU Secrets failure from what the CLI reports: run secrets doctor, act on each warn and fail (what an agent fixes, what goes to the operator), stop on an identity problem (LINKED_MAPPING, MAPPING_CHANGED, OUTSIDE_PROJECT, DUPLICATE_PROJECT, an unknown identity, the parent-identity warning), read secrets whoami, identity list, project list, store list, and store sync, add a missing project with project add, and read a refusal of a store write by the secret scan"
---

> [!important] THIS FILE IS AN INSTRUCTION. WHEN REFERENCED IT IS MEANT TO BE TAKEN AS AN ACTION.

Run `flint shard start nsec` if you haven't already.

# Skill: Diagnose

Find what NUU Secrets reports, fix the parts that an agent may fix, and send the other parts to the operator. Report what the CLI says. Do not guess a cause that the output does not show. No step of this skill shows a value.

# Input

- The failure: the command, its exit status, and its `✘` message (also the output of a store write that the operator sends you)
- The project folder (it has `secrets.toml`, or a parent folder has it), or no project
- (Optional) The environments that the task needs

# Actions

1. **Read the rules.** You read [[dev-knw-nsec-rules]]. Never run `sops`, never open the key file, and never open a store file to diagnose. If the input is a refused store write, a cleanup warning, or a `Nothing was pushed.` message, skip steps 2–8. Follow step 9, then step 10. Use plain `secrets doctor` with the same store, mapping, and environment options. Do not run `store sync`, `doctor --online`, or another command that refreshes this clone before the operator approves the repair. If the shell cannot find `secrets` (`command not found`), stop. Give the operator the install command of [[dev-knw-nsec-cli]] § Install. Do not install the CLI yourself. On a machine that has no key or no store yet, run `secrets setup --print` (it changes nothing), and follow [[dev-knw-nsec-setup]]: give the operator the operator commands of each `todo` step.
2. **Read the cached state first, then refresh.** Run plain `secrets doctor` with the mapping, store, and environment options for this task. Keep its lines about unpushed commits, changed files, and links. Use `secrets store list` to match the mapping URL to its local store name. When the network is available, refresh only that store:
   ```sh
   secrets store sync <local store name>
   ```
   It fetches that clone and resets it to the remote. It also reports recipient changes (step 8). Then run plain `secrets doctor` with the same options. If the task covers more stores, check each store before you refresh it. Do not use an unnamed sync after a doctor that checked only one store.
3. **Run the checks.** In the project folder, run:
   ```sh
   secrets doctor
   ```
   - Without `--online`, `doctor` reads the cached clones, makes no network call, and changes nothing. The line `cached revision <revision> (<date>): doctor reads the clone offline, so it can be older than the remote` gives the cached commit and its commit date. It does not give the last refresh time.
   - `secrets doctor --online` first fetches each checked clone and resets it to the remote, then checks. The reset discards the local changes and commits of an interrupted command (the clone is a cache that the CLI owns). It never clones a missing store. Run it only after the plain check with the same options (step 2). It does not show the changes of `recipients.toml`: use `secrets store sync <local store name>` for a checked store.
   - Outside the project folder, add `--mapping <path>` or `--project <name>`.
   - Add `--env <env>` (one or more times) to check only some environments. The default is `dev`, `preview`, and `prod`.
   - The exit status is 1 when a check fails. The last line counts the results. An exit status 0 can still have `warn` and `skipped` lines.
4. **Act on each `fail`, `warn`, and important `skipped` line.** A check can have a remedy line (`→ …`). Use this table to decide who acts. "Operator" means: do not do it yourself; put it in the report of step 10.

   | Section | Message (start) | Who acts | Action |
   |---|---|---|---|
   | Tools | `ok  gitleaks <version>: the CLI scans each store commit with it` | — | Normal. It proves only that `gitleaks version` starts, not that each flag of the scan works. |
   | Tools | `warn  gitleaks is not on PATH: the CLI does not scan store commits` | Operator | A store write on this machine has no secret scan. Give the remedy line (the install command). Ask the operator to install gitleaks. |
   | Tools | `warn  gitleaks does not start: a write that needs a commit stops (SECRET_SCAN_FAILED)`, or `warn  gitleaks does not start` | Operator | Report it. Do not work around it: do not remove, rename, or hide gitleaks on `PATH`, and do not set `SECRETS_GITLEAKS=0`. |
   | Tools | `warn  SECRETS_GITLEAKS=0: the CLI does not scan store commits` | Operator | An inherited switch turns off the secret scan. Report it. Ask the operator to unset it. Do not run a store write while it is set ([[dev-knw-nsec-rules]] § Rule 10). Agents never set it. |
   | Tools | `fail  sops <version> is too old (the CLI needs 3.11 or later for set --value-stdin and encrypt from stdin)` | Operator | Give the remedy line (`install or upgrade to SOPS 3.11 or later: …`). Do not install or change SOPS yourself. |
   | Tools | another tool: `<tool> is not on PATH`, `<tool> does not start` | Operator | Give the remedy line (the install command). |
   | Key | `ok  key selection: <reason>` | — | Normal. It names the identity (or the key file) that the CLI selected, and why ([[dev-knw-nsec-model]] § Identities and the key selection). When access is not correct, put this line in the report. Do not select another identity yourself. |
   | Key | `fail  There is no identity named "<name>" (<where>)…` with ``→ run `secrets identity list`, then select an identity that exists …`` | Operator | Stop (`UNKNOWN_IDENTITY`). Run `secrets identity list`. Give the operator both outputs. The remedy commands are operator commands ([[dev-knw-nsec-rules]] § Rule 12). |
   | Key | a `fail` of a key file with ``→ run `secrets identity list` to check the key file of the identity <name>`` | Operator | The key file of the selected identity is missing or cannot be used. Run `secrets identity list` and give the row of that identity (name, key file path, state). Do not use step 1a of the key request ([[dev-sk-nsec-request_value]], step 4). |
   | Key | `fail  no age key file at <path>`, `fail  the age key file <path> is not a regular file`, `fail  the age key file <path> cannot be read` | Operator | Use the key request of [[dev-sk-nsec-request_value]] (step 4): step 1a for a missing key file with no other key source, step 1c for a file that exists but cannot be used. Do not ask the operator to replace an existing file with `secrets keygen`. Do not make a key. |
   | Key | `warn  no age key file at <path>; SOPS can decrypt with <source>` | — | SOPS has another key source (`SOPS_AGE_KEY`, `SOPS_AGE_KEY_CMD`, or the SOPS default key file). Nothing to fix for SOPS. Report it once. Do not read `SOPS_AGE_KEY`, and do not run the command of `SOPS_AGE_KEY_CMD`. |
   | Key | `skipped  the public key of SOPS_AGE_KEY_CMD (the CLI does not run the command)` | — | The key of this machine is unknown to `doctor`. See the `unknown` rows. |
   | Key | `key file <path> has mode <mode>: other users can read it` | Operator | Give the remedy (`chmod 600 <path>`). Do not change the key file. |
   | Key | another `fail` | Operator | Give the message. |
   | Store | `this machine is not a recipient of the store` | Operator | Look at the Key section first. When it has a `fail`, use that row. A doctor public key can come from another SOPS source. Use [[dev-sk-nsec-request_value]] (step 4). Use `secrets keygen --public` only without `--file`. If it fails, follow step 1c of that request. The operator does not make a new key. An admin adds it with `secrets recipient add`. |
   | Store | `warn  this machine as a recipient: unknown (SOPS_AGE_KEY_CMD gives the key, and doctor does not run it)` | — | This is not proof that this machine is not a recipient. A `secret <cmd>` run shows if the key decrypts. Report it. Do not request a new key. |
   | Mapping | `skipped  <env>: the access is unknown (doctor does not run SOPS_AGE_KEY_CMD)` | — | The same as the row above. |
   | Store | `ok  cached revision …: doctor reads the clone offline, so it can be older than the remote` | Agent | Normal. After the plain check, run `secrets doctor --online` with the same options. For recipient changes, use `secrets store sync <local store name>` for a checked store, then plain `doctor`. |
   | Store | `fail  --online: cannot refresh the clone: …` and `warn  cached revision …: the refresh failed, …` | Agent, then operator | Give the exact line. The cause can be the network, the remote, the local clone, or the access. Retry once. Then report. |
   | Store | `fail  <file> is a symbolic link (<path>)`, `fail  <file> is committed as a symbolic link (mode 120000) (<path>)`, `fail  <file> is committed as a folder (<path>)`, `fail  <file> is not a regular file (<path>)`, `fail  <file> cannot be checked (<code>) (<path>)` | Operator | A store file is a link or a special file, or the CLI cannot check it (`<code>` is a system error code). `doctor` then skips the other checks of that store. Give the lines. The repair is an operator task: for a link, in a checkout of the store, `git rm` the link, commit, push, then `secrets store fix-config`; for `cannot be checked`, the access of the user to the clone on this machine. Do not change the clone or its modes. |
   | Store | `✘ The stores folder <path> is a symbolic link …`, `✘ The CLI cannot check the stores folder <path> (<code>). …`, `✘ The clone folder of store <name> is not safe to use: …` | Operator | Give the message. Do not change modes, and do not move, delete, or link folders in the secrets home. |
   | Store | `.sops.yaml does not match recipients.toml: …` | Operator | Give the line. `doctor` shows that the two files do not agree. It does not show which file changed. Do not run `secrets store fix-config`. |
   | Store | `warn  .gitleaks.toml is missing`, `warn  .gitleaks.toml differs from the text that the CLI generates` | Operator | Give the exact command of the remedy line: `secrets store fix-config --store <name>`. It is an operator command: never run it yourself. Each existing store needs it once after cycle H. Tell the operator to run it once for each store, on one machine, with the build of `secrets` canon `35ecaba` or a later build: an older build writes the old text again ([[dev-knw-nsec-cli]] § The config of stores). The secret scan of the CLI still works before the repair. |
   | Store | `ok  .gitleaks.toml allows only the recipient key lines of recipients.toml` | — | The configuration matches the generated text. This line is not a result from a gitleaks scan. |
   | Store | `fail  <file>: line <n>: the SOPS metadata field "<field>" does not have the form that SOPS writes` | Operator | A store file breaks the store policy. The line names the field, never its value. Give the line and its remedy line. Do not open the file. A store write to this store stops (`UNSAFE_STORE`). |
   | Store | a problem of a store file, or `group <group> has no recipient` | Operator | Give the message. |
   | Store | `the store has no recovery key` | Operator | Give the remedy line. |
   | Store | `a stale lock of process <pid>` | Agent | The next CLI command removes it. Run your command again. Do not delete the lock file. |
   | Store | `<n> unpushed commit(s) … from an interrupted command` | Operator | A write of the operator stopped. The operator runs that command again. `doctor --online` and `store sync` discard such commits: report the line before you run them. |
   | Store | `the clone has no commit`, `The clone of store <name> is missing` | Operator | Give the message and the secrets home. Do not delete or edit a clone. |
   | Store | `the store of the mapping is not cloned on this machine` | Agent | When the task allows network access, run the app with `secret <cmd>` (without `--offline`) or run `secrets store clone <url>`. `doctor --online` does not clone. First check that the secrets home is the intended one ([[dev-knw-nsec-model]] § The store). |
   | Stores | `there is no store on this machine` | Operator | With no mapping, the operator decides which store to clone or create. |
   | Mapping | `<env>: <VAR> → secret://<name> is not in the store` | Operator | Request the value with [[dev-sk-nsec-request_value]]. For `preview`, a dev value is enough. |
   | Mapping | `<env>: no access (this machine is not in group <group>)` (skipped) | — | Normal when this machine does not need that environment. Report it only when the task needs that environment. |
   | Mapping | `<env>: sops decrypt failed …` | Operator | SOPS did not decrypt the file with the available key sources. This error does not prove that the selected key file is valid. Give the line. See the Key and Store rows. |
   | Mapping | `<VAR> goes to convex: a NEXT_PUBLIC_ variable is for the browser build` | Agent | Remove only `convex` from the targets of that entry. Keep the rest. See step 5. |
   | Mapping | `<VAR>: the literal in <file> looks like a secret` | Operator | A value can be in a committed file. Do not repeat it. Follow [[dev-knw-nsec-rules]] § If a value leaks. |
   | Mapping | `<path> is not valid:` with a list of problems | Agent | Fix each named entry of `secrets.toml` or `secrets.local.toml` with `secrets bind` or `secrets unbind` (`--local` for the local file). The problems name fields, not values. For a `targets.<target>: …` problem, fix the table with `secrets target set` only when the operator or the task gives its content (Rule 8); else report it. When a command refuses the form of the file (`MAPPING_EDIT`), report it to the operator. A table that is not valid stops every command of the project, also `secret <cmd>`, `run`, and `export`. |
   | Mapping | `env.<VAR>.to: "convex" is not allowed: the Convex tool owns <VAR>, …` | Agent | Remove `convex` from the targets of that entry (step 5 shows the forms). The Convex tool writes that variable itself. |
   | Project hygiene | `.gitignore does not cover …` | Agent | Run `secrets project gitignore` in the project folder. Do not edit `.gitignore` by hand. |
   | Project hygiene | `<file> has names that secrets.toml manages: <VAR>, …` | Operator | A `.env*` file has duplicates. Do not open the file. The operator removes the names. |
   | Project hygiene | `secrets.local.toml is tracked by Git` | Agent and operator | Run `git rm --cached secrets.local.toml` and cover it in `.gitignore`. Tell the operator: the Git history keeps the old file. |
   | Projects | `warn  the path of project <name> is missing: <path>` | Operator | Report it. The remedy `secrets project forget <path>` is an operator command: the entry can hold the project identity. |
   | Projects | `warn  the entry <path> of project <name> is for the same secrets.toml as another entry (identity <name>)` | Operator | Report it with the remedy line (`secrets project identity <name> --mapping <path>` or `--clear`). A command of that project stops with `DUPLICATE_PROJECT` when the identities are different. |
   | Projects | a `warn` that `projects.toml` is not valid | Operator | Report it. Do not edit or delete the file. |
   | Mapping | an identity error (`LINKED_MAPPING`, `MAPPING_CHANGED`, `OUTSIDE_PROJECT`, `DUPLICATE_PROJECT`) | Operator | Stop. See step 6. |
   | Machine | `full-disk encryption: …` | Operator | Report it once. It does not stop the work. |

   A command that refreshes first (`secret`, `export`, `plan` without `--offline`, `push`, `store sync`, `doctor --online`) resets the work tree of the clone. It can remove a local link and then succeed. Report each `doctor` line about a link also when a later command succeeds.

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
   - Run `secrets doctor` again. The `warn` for this variable must be gone.

6. **Read the identity, and stop on an identity problem.** The CLI selects one key file for each command ([[dev-knw-nsec-model]] § Identities and the key selection). Use that selection: do not add `--identity`, and do not set `SECRETS_IDENTITY` or `SOPS_AGE_KEY_FILE`, unless the operator names the identity ([[dev-knw-nsec-rules]] § Rule 12).
   - **Stop on these problems.** When a command or `doctor` reports `LINKED_MAPPING`, `MAPPING_CHANGED`, `OUTSIDE_PROJECT`, `DUPLICATE_PROJECT`, `UNKNOWN_IDENTITY` (`✘ There is no identity named …`), `PROJECT_INDEX` (`✘ The CLI cannot read the identity of project …`), the warning `secrets: warning: <path> has no identity of its own in projects.toml, so the CLI uses …`, or the warning that the identity of a link target does not apply, stop the work that needs the key. Copy the exact message and its remedy lines (`Next:`, `Or:`, the other hint lines, or the `→` line) into the report of step 10. The message starts are in [[dev-knw-nsec-cli]] § Messages that agents see. Do not apply a remedy yourself: no `--identity`, no `secrets project identity <name>` or `--clear`, no `secrets project forget`, no change of a link, and no run from another folder or with another `--mapping` to pass the check. The operator decides. In a headless session, this is a blocker ([[dev-hinit-nsec]]).
   - **Collect the public facts for the report.** Run `secrets identity list` (the identities, their public keys, the key file paths and states, the stores and groups of each identity, and the `active:` line), `secrets project list` (the project index and the state of each entry), and `secrets project identity` in the project folder (the project identity; it changes nothing). None of them shows a private key or a value. Do not open a key file that they name.
   - **`secrets project list` states.** `ok`: normal. `missing`: the path does not exist (the operator can run `secrets project forget <path>`). `linked`: the path has a symbolic link, so its identity does not apply (the operator can run `secrets project identity --clear --mapping <path>`). `duplicate`: another entry is for the same file (the operator runs `secrets project identity <name> --mapping <path>` or `secrets project forget <path>`). Each remedy is an operator command.
   - **A missing project.** When `secrets project list` is empty or does not show a project of the task, add it to the project index. First run `secrets project add <folder> --dry-run` (the project folder, or a folder above it, for example a monorepo). Read the lines (`add`, `update`, `known`, `skipped <reason>`). Then run the same command without `--dry-run`. It writes only `projects.toml`: it reads no store and no value, and it sets no identity ([[dev-knw-nsec-cli]] § `secrets project add`). Report each `skipped` line. For `duplicate` and `linked`, give the operator the remedy of the states above. When the search stops at a limit, give a smaller folder.
   - **No identities.** When `secrets --help` lists no `identity` command, or `secrets identity list` says `No identities (…)`, the CLI selects `SOPS_AGE_KEY_FILE`, else the default key file, as before.

   Run `secrets whoami` when the problem is access.
   - An `identity` row `<name> (<reason>)` names the selected identity. A `key` row ``the identity selection is not valid (run `secrets identity list`)`` is an identity problem: stop as above.
   - A `key` row with ``no key (run `secrets keygen`)`` means that the CLI cannot read a public key from the key file. The file can be missing, not readable, or not valid. Run `secrets doctor` and use its Key line.
   - A `store` row `<name>  <id> in groups <groups>` gives the groups of this machine. A machine in the group `dev` only cannot open `prod` or `deploy`.
   - A `store` row `<name>  not a recipient` means that whoami found no matching public key from the selected CLI key file. It does not check all SOPS key sources. Use doctor before you infer missing access.
   - A `store` row ``<name>  cannot read the clone (run `secrets doctor`)`` means that a clone file cannot be read. Run `secrets doctor`.
   - The `Account` part is optional. "Not logged in" and ``Run `secrets login` to authenticate.`` are not a problem. No command needs a login, and `login` is an operator command.
   - For the full public key of this machine, run `secrets keygen --public`. It makes no key.
7. **List the store clones.** Run `secrets store list`.
   - Each row is `NAME`, `REVISION`, `URL`.
   - `missing` means that the clone folder is gone. `invalid clone folder` and `invalid entry` mean a damaged clone or a damaged `stores.toml`. These go to the operator. Do not edit `stores.toml`.
   - `secrets: there is no store on this machine` means that no store is cloned in this secrets home.
   - A folder `.removing-<name>-<random>` in the stores folder is the rest of a failed `store remove`. Report it to the operator. Do not delete it.
   - An unwanted clone (for example in the wrong secrets home) goes to the operator: only the operator runs `secrets store remove <name>`.
8. **Read the output of `secrets store sync <local store name>`.**
   - `<store>  up to date @ <revision>` or `<store>  <old> → <new>`: normal.
   - `secrets: store <name> had <n> unpushed commit(s) from an interrupted command. They are removed; …` or `secrets: store <name> had changed files from an interrupted command. They are removed.`: the refresh reset the clone. Report it. Any refresh can print these lines (`secret`, `export`, `store sync`, `doctor --online`).
   - `The history of the remote changed (a forced push).`: tell the operator.
   - `recipients.toml changed in <n> commit(s) since the last sync:` with `+`, `-`, or `~` lines: a recipient was added, removed, or changed. Copy these lines (ids, groups, fingerprints, and authors are public) into your report for the operator.
   - `secrets: store <name>: <message>` and `<n> of <m> store(s) did not sync.`: the sync of that store failed. Give the exact line. The cause can be the network, the remote, the local clone, a link below the secrets home, or the configuration. Retry once. Then report.
9. **Read a refusal of a store write.** A store write is an operator command. You see its output when the operator sends it to you, or when you run an exact write command that the operator gave you. Use [[dev-knw-nsec-rules]] § Rule 10 and [[dev-knw-nsec-cli]] § The secret scan of store commits. The text output does not print the code: identify it from the start of the message.
   - `✘ The secret scan (gitleaks) found <n> possible secret(s) …` (`SECRET_SCAN_FOUND`): stop and report `SECRET_SCAN_FOUND`. Report only the safe count, rule labels, and file labels from the CLI. Preserve any cleanup warning. Do not open the matching line. Do not bypass or repeatedly retry the finding. A finding in an encrypted file can indicate plaintext metadata, modified tooling, or a false positive. Run `secrets doctor` and report the result. Only the operator decides whether a finding is a false positive.
   - `✘ The secret scan (gitleaks) of the change to store <store> did not complete: <reason>.` (`SECRET_SCAN_FAILED`), with `Next: secrets doctor` and `Then run the command again. Do not bypass a failed scan.`: run `secrets doctor` and report the fixed failure reason. Preserve any cleanup warning. Do not bypass a failed scan.
   - `✘ A file of store <store> breaks the store policy. …` (`UNSAFE_STORE`) or `✘ The result of a SOPS write breaks the store policy. …` (`UNSAFE_RESULT`), with a line such as `- dev.enc.yaml: line <n>: the SOPS metadata field "<field>" does not have the form that SOPS writes`: report the file label, the line number, and the field name. Do not retry the write. Do not open the store file. The hint ``Check the SOPS version (`sops --version`).`` is for the operator.
   - A message with `Nothing was pushed.`, the line `A scan refusal before staging writes no new Git object. …`, or a cleanup line such as `The CLI could not clean the clone of store <name>. …`: report it. A later failure can leave local objects. Do not assume that a reset removed them. Run the repair (`secrets store remove <name>`, then `secrets store clone <url>`) only when the operator approves it.
   - `secrets: gitleaks is not on PATH, so the secret scan of the store commit did not run. …`: the scan did not run. Check the command's exit status. If the command succeeded, the write was made without a scan. Report both facts. Ask the operator to install gitleaks.
   - Another fixed error of a store write (`UNSAFE_FILE`, also `… has a file that the CLI cannot check …`, `COMMIT_FAILED`, `WRITE_FAILED`, or `COMMIT_MISMATCH`; the message starts are in the table of [[dev-knw-nsec-cli]] § The messages of a store write): give the fixed text to the operator. Do not change the store clone.
   - Never set `SECRETS_GITLEAKS=0`, and never remove, rename, or hide gitleaks on `PATH` to pass the write. Before you run a write command that the operator gave you, check for an inherited `SECRETS_GITLEAKS=0` ([[dev-knw-nsec-rules]] § Rule 10).
10. **Report.** Write one report with:
    - What you fixed, and the `doctor` result after your fixes.
    - Each item for the operator: the message, the exact command for the operator, and why.
    - Never a value. Names, ids, public keys, fingerprints, paths, and revisions are public.
    - For a store write: copy the fixed write-error text, safe rule labels, safe file labels, field names, and cleanup warnings. Do not copy file content or raw tool output. Some error lines show a path that the user chose or set. The output rules of the CLI do not cover the text of that path. Do not copy a line when its path or another field holds a value: name the kind of error.

# Output

- What the CLI reports about the failure (observations, not guesses)
- The agent fixes done, with a `secrets doctor` result after them
- A report for the operator with each remaining item and its exact command
