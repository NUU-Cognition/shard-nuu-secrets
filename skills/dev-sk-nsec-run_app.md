---
description: "Run a program with the values of the store: secrets pull --check, then secrets pull when the block of .env.local is not current, then the normal dev command; read the pull line, the expiry warnings, and the exit status, act on a refusal (a managed name outside the block, a changed block, a .env.local that Git does not ignore) and on a moved or removed ref; never read or change the block"
---

> [!important] THIS FILE IS AN INSTRUCTION. WHEN REFERENCED IT IS MEANT TO BE TAKEN AS AN ACTION.

Run `flint shard start nsec` if you haven't already.

# Skill: Run an App with Secrets

Run a program (a dev server, a test, a script) with the variables of its project from the store. `secrets pull` writes the values of the target `local` into one block of the `.env.local` file of the project. Then you run the normal command of the program (`pnpm dev`, `next dev`, `vite`, `npx convex dev`): the program reads `.env.local`. The direction is one way: store → `.env.local`. To change a value, the operator changes the store, then you pull again.

The block is the lines between the start marker `# >>> nuu-secrets · …` and the end marker `# <<< nuu-secrets <<<`. The CLI owns these lines. You never read, print, copy, or change them ([[dev-knw-nsec-rules]] § Rule 2).

# Input

- The program and its arguments, for example `pnpm dev` or `pnpm test`
- The project folder (it has `secrets.toml`, or a parent folder has it)
- (Optional) The environment: `dev` (the default) or `preview`
- (Optional) Network state: offline or online

# Actions

1. **Check the rules.** You read [[dev-knw-nsec-rules]]. The program must not print its process environment or a value. Do not start it with debug or dump options. Never open, print, or search `.env.local` (also not with `cat`, `grep`, `head`, `diff`, or an editor tool).
2. **Go to the app folder.** Run `secrets pull` in the folder of the `secrets.toml`, or in a child folder. When you cannot, add `--mapping <path>` or `--project <name>` (not both). For each project of the current Git repository at once, use `--all` (step 9).
3. **Select the environment.**
   - Use the default `dev` for development and tests.
   - Use `--env preview` only when the task needs preview values. `preview` reads only `preview.enc.yaml`: a required secret with no preview value stops the pull, also when dev has a value. Request the preview value ([[dev-sk-nsec-request_value]]). Do not use `dev` in its place.
   - `secrets pull` refuses `--env prod` and `--env deploy` (`✘ secrets pull writes only env dev or preview, not <env>. Production values do not go to the disk.`, `PULL_ENV`). Production values go to a platform with `secrets plan` and `secrets push` ([[dev-sk-nsec-plan_push]]). Do not try another command to get them.
4. **Check the block first.**
   ```sh
   secrets pull --check
   secrets pull --check --env preview
   ```
   - `--check` writes nothing. It needs the key and reads the store, as a pull does. stdout has one line for each project, then a summary:
     - `current  <project> · env <env> · store <revision> (<date>) · .env.local`: the block is current. Go to step 6.
     - `stale    <project> · env <env> · .env.local`, with one `- <reason>` line for each reason: `.env.local does not exist`, `.env.local has no block of NUU Secrets`, `the block was changed after the last pull: <VAR>, …`, `the block is for env <env>`, `the block is from store <revision> (<date>); the store is at <revision> (<date>)`, or `names that are not the same as the store now: <VAR>, …`. A refusal of step 7 also shows as a reason. The names are public. The lines never show a value.
     - `pull --check: <n> current, <n> not current. Nothing was written.` The exit status is 1 when a block is not current.
   - Use `--offline` when there is no network, or when the task forbids network access (step 5).
5. **Pull when the block is not current.**
   ```sh
   secrets pull
   secrets pull --env preview
   secrets pull --offline
   ```
   - Do not add `--force` yourself. It replaces a block that was changed by hand (step 7).
   - `--offline` uses the cached clone, names its revision, and makes no network call. When the store has no cached clone, it stops with `✘ --offline: the store of <project> (<url>) has no cached clone on this machine. Nothing was cloned.` (exit 1). Then:
     - when the task allows network access, run `secrets pull` once without `--offline` (it clones the store), or run `secrets store clone <url>`;
     - else report a blocker.
   - Read the `pull` line on stderr. It has names and counts, never values:
     `secrets: pull <project> · env <env> · store <revision> (<date>) · <n> variables → .env.local`
     - Check the project, the environment, and the count.
     - `· <n> optional not set` means that the store has no value for `<n>` `optional` entries. The block does not have these names.
     - `· <n> not declared in <env>` means that the secrets of `<n>` entries do not declare the environment. The block does not have these names, and it is not an error. When the app needs one of them, tell the operator: the operator changes the set ([[dev-knw-nsec-rules]] § Rule 13).
     - `store not read (no ref entries)` means that the mapping has only literals for the target `local`.
     - `(unchanged)` at the end means that the bytes of the file did not change and the file has mode 600. The CLI wrote nothing, so a dev server does not restart. When only the mode was not 600, the CLI replaced the file with the same bytes at mode 600 (no `(unchanged)`): a dev server can restart.
   - Read the warnings on stderr:
     - `secrets: warning: <VAR> (secret://<name>, <env>) expires on <date> (in <n> days). …` or `… expired on <date> (<n> days ago). …` is an expiry warning. The pull still writes the block. Tell the operator the variable, the secret name, the environment, and the date. Do not run the `secrets expiry set` command of the line: it is for the operator ([[dev-knw-nsec-rules]] § Rule 13). When the program then fails with an authentication error, report the warning with the failure.
     - `secrets: warning: <VAR> → secret://<name> (optional, not set): moved to …` (or `removed on …`): the block does not have that variable. Follow [[dev-sk-nsec-diagnose]] (step 7) after the run.
     - `secrets: warning: <VAR> → secret://<name> has a value in env <env>, and <name> does not declare env <env> … The value is not used. …`: the block does not have that variable. Report the warning with its two commands to the operator; do not run them ([[dev-knw-nsec-rules]] § Rule 13).
     - `secrets: warning: <path> is in a folder that iCloud Drive syncs. iCloud keeps a copy of the plaintext values. …`: the pull wrote the file. Tell the operator. Do not move the project yourself.
     - `secrets: warning: the dotenv npm package loses the variable __proto__ when it reads .env.local. …`: tell the operator. The operator renames the variable.
     - `secrets: cloning the store of <project> from <url>` means a first clone (a network access). Check that the secrets home is the intended one ([[dev-knw-nsec-model]] § The store). Only the operator can remove a clone (`secrets store remove`).
6. **Run the normal command.** Run the program as the project documents it, with no prefix: `pnpm dev`, `next dev`, `vite`, `npx convex dev`, `pnpm test`.
   - The program must read `.env.local`. Next.js, Vite, and the Convex CLI read it. When a program does not read it (for example plain `node server.js`), use the loader of the program with the path only: `node --env-file=.env.local server.js`. Never load the file into your own shell (`source .env.local`, `. .env.local`, `export $(cat .env.local)`, `set -a`): then the values are in your shell.
   - A long-running program (a dev server) does not stop. Run it in the background of your harness. Stop it when the task is done.
   - When the store changes during the work (the operator writes a new value), run `secrets pull --check` and `secrets pull` again. A dev server can restart when `.env.local` changes.
   - When the program is `turbo run …`, Turborepo starts the dev command of each app, and each app reads its own `.env.local`. Pull each project first: `secrets pull --all` in the repository (step 9).
7. **Act on a refusal of the pull.** A refusal exits with status 1 and writes nothing. The error starts with `✘` and ends with `Nothing was written.`
   - **A managed name outside the block** (`PULL_MANAGED_OUTSIDE`): `✘ <path> sets names that secrets.toml gives to local, outside the block of NUU Secrets:` with one line `  - line <n>: <VAR>` for each name. Stop. Report each line number and name to the operator. Do not delete, comment out, or move the line, and do not remove the target `local` of the name, without the approval of the operator ([[dev-knw-nsec-rules]] § Rule 2). The operator removes the line, or removes the target `local` of the name in `secrets.toml` (a decision of the operator; for a name that another tool writes, for example `NEXT_PUBLIC_CONVEX_URL` from `npx convex dev`, the usual fix is to remove `local` from its entry). After the change of the operator, pull again.
   - **A changed block** (`PULL_CHANGED_BLOCK`): `✘ The block of NUU Secrets in <path> was changed after the last pull (lines <n> to <n>). The lines that are not the same as the store now: <VAR>, ….` A person or a program changed a line of the block. Do not open the file, and do not add `--force` yourself. Tell the operator the names. When a value must change, the operator changes the store (`secrets set <name> --env <env>`). With the approval of the operator, run `secrets pull --force`: it replaces the block.
   - **A broken block** (`PULL_BROKEN_BLOCK`): `✘ The block of NUU Secrets in <path> is broken: <problem>.` (a start marker that is not valid, no end marker, an end marker with no start marker, or two blocks). Do not open or edit the file. Report the message to the operator. The operator removes the lines of the broken block, then you pull again.
   - **Git does not ignore `.env.local`** (`PULL_NOT_IGNORED`): `✘ Git does not ignore <path>. …` with the hint `Next: secrets project gitignore`. Run `secrets project gitignore` in the project folder (an agent command), then pull again. Commit the `.gitignore` change with the other project files.
   - **Git tracks `.env.local`** (`PULL_TRACKED`): `✘ Git tracks <path>. A pull would put plaintext values into a commit.` Stop. Tell the operator. The fix is `git rm --cached .env.local`, then `secrets project gitignore`, and the Git history keeps the old file: the operator decides. Do not run `git rm --cached` without the approval of the operator.
   - **A symbolic link or a file that is not a regular file** (`PULL_LINK`, `PULL_FILE`): `.env.local` is a link, the path of `secrets.toml` has a folder link, `.env.local` is not a regular file, or it is not valid UTF-8 text. The CLI never writes through a link. Do not remove or change the link. Report the message to the operator. For a folder link, run the command again from the real folder of the project only when the operator says so.
   - **The folder or the file changed during the pull** (`PULL_FOLDER_CHANGED`, `PULL_FILE_CHANGED`): another program changed the project folder or `.env.local` (for example `npx convex dev` wrote it). Run `secrets pull` again one time. When it fails again, report it.
   - **A value that the block cannot hold** (`BAD_VALUE`): the error names the variable, never the value. Tell the operator. The operator changes the value in the store.
   - **`--all`** (`PULL_REFUSED`): `✘ secrets pull stopped before the first write: <n> problem(s).` with one line for each project. Act on each line as above. No project got a write: the checks of all projects come before the first write. A refusal during the writes (`PULL_FILE_CHANGED` or `PULL_FOLDER_CHANGED` of a later project) stops the pull, but the projects before it stay written: their `pull` lines are on stderr. Report those projects as written, act on the refusal, and run `secrets pull --check --all` (names only).
8. **Act on the other failures.** The resolve of the values is the same as before a pull:

   | Status | Meaning | Action |
   |---|---|---|
   | `0` | The block is written (or `(unchanged)`). | Run the program (step 6). |
   | `1` with `✘ <n> variable(s) have no value in env <env> of store <store>:` | Values are missing. Nothing was written. | Use [[dev-sk-nsec-request_value]] for each listed name with no hint. |
   | `1` with `✘ <n> variable(s) have no value …` and a line ``- <VAR> → secret://<name>: moved to secret://<new name> …; run `secrets refs fix` `` or `… removed on …` | The change log moved or removed the name. Nothing was written. The CLI never uses the new name. | Do not request a value for that name. Use [[dev-sk-nsec-diagnose]] (step 7): `secrets refs fix --dry-run`, then ask the operator. |
   | `1` with `✘ There is no secrets.toml …` | No mapping here. | Go to the project folder, or use [[dev-sk-nsec-setup_project]]. |
   | `1` with `✘ <path> is not valid:` | A mapping problem. A `[targets.*]` table that is not valid also stops `secrets pull`. | Run `secrets doctor` and use [[dev-sk-nsec-diagnose]]. |
   | `1` with `✘ No usable age key: …` | The key file is missing, not a regular file, or cannot be read. Nothing was written. | Tell the operator the path. Do not make a key. Use [[dev-sk-nsec-request_value]] (the key request). |
   | `1` with an identity error: `✘ There is no identity named …`, `✘ The CLI cannot read the identity of project …`, or the errors `LINKED_MAPPING`, `MAPPING_CHANGED`, `OUTSIDE_PROJECT`, `DUPLICATE_PROJECT` ([[dev-knw-nsec-cli]] § Messages that agents see) | The CLI cannot use the identity of the project, and does not use the default in its place. Nothing was written. | Stop. Use [[dev-sk-nsec-diagnose]] (step 6). Do not add `--identity`, and do not run from another folder to pass the check. |
   | `0` with `secrets: warning: … has no identity of its own in projects.toml, so the CLI uses …` | The pull used the default identity, not the identity of the parent project. | Stop before the program runs, and report the warning ([[dev-sk-nsec-diagnose]], step 6). |
   | `1` with `✘ The store file <file> of store <store> has SOPS metadata that the CLI does not decrypt. SOPS did not run, and nothing was read.` (`UNSAFE_STORE`) | The store file has a key of another system, an unknown metadata key, a YAML problem, or no SOPS metadata. SOPS did not run. Nothing was written. | Stop. Do not open, edit, or decrypt the file. Give the operator the exact message and its lines ([[dev-sk-nsec-diagnose]], step 4). |
   | `1` with `✘ sops decrypt failed …` | SOPS did not decrypt the file with the available key sources. This error does not prove that the selected key file is valid. | Use [[dev-sk-nsec-diagnose]]. Do not run `sops`. |
   | `1` with `✘ --offline: … has no cached clone …` | `--offline` and no clone. Nothing was written. | See step 5. |
   | `1` with `✘ The stores folder <path> is a symbolic link …`, `✘ The CLI cannot check the stores folder <path> (<code>). …`, `✘ Store <name> has a file that is not a regular file …`, or `✘ Store <name> has a file that the CLI cannot check …` | An unsafe store layout, or a file or folder that the CLI cannot check. Nothing was written. | Report the message to the operator. Do not change modes or folders in the secrets home. Use [[dev-sk-nsec-diagnose]]. |
   | `1` with `Unknown command: run` | An old instruction used the `run` command of an older CLI. Since `0.4.0-beta.0`, the CLI has no `run` command and no alias for it. | Use this skill: `secrets pull`, then the normal command. |
   | `1` with `error: …` | The option parser refused an option (for example `error: unknown option '--bad'`). | Fix the option. |

9. **Many projects in one repository.** `secrets pull --all` (and `secrets pull --check --all`) acts on each project of the project index in the current Git repository. It checks each project before the first write: when the checks find a refusal in one project, it writes no file. Then it writes the projects one by one. A refusal during the writes (`PULL_FILE_CHANGED` or `PULL_FOLDER_CHANGED` of a later project) stops the pull, but the projects before it stay written: their `pull` lines are on stderr. When it stops with `✘ secrets pull --all works in a Git repository. …` (`PULL_NO_REPOSITORY`), run it in the repository, or pull in each project folder. When it stops with `✘ The project index has no project in <root>.` (`NO_PROJECTS`), run `secrets project add <folder> --dry-run`, then `secrets project add <folder>` ([[dev-knw-nsec-cli]] § `secrets project add`), or pull in each project folder.
10. **Do not put `secrets pull` in project files.** Do not add it to the scripts of `package.json` (for example `predev`), to a `Makefile`, or to CI files. CI has no key, and a project file with `secrets pull` fails on a machine without NUU Secrets. The person or the agent runs `secrets pull` before the dev command.
11. **Check a variable without its value** when the task needs proof. Use the safe signals: the `pull` line (the count and `optional not set`), `secrets pull --check` (`current`), and `secrets doctor` (it names each required secret name that has no value, and checks the block). A required `ref` with no value stops the pull, so a successful pull proves that each required variable of the target `local` is in the block. Never read `.env.local` to check a value.
12. **If a value shows** in the output of the program or of a command, follow [[dev-knw-nsec-rules]] § If a value leaks.

# Output

- A current block in `.env.local` (`secrets pull --check` says `current`), and the program ran with its normal command; or a clear failure with its next step
- No value in your output, in a file that you wrote, or in a command argument, and no change of the block or of a line of `.env.local` by you
