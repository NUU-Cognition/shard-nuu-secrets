---
description: "Help the operator migrate a store of the previous version (store format 1) to NUU Secrets v2: read the guide, check the versions, run only the read-only steps (doctor, secrets store migrate --plan, a read of the plan file, refs fix --dry-run, pull --check), report the conflicts and the preview list, and give the operator each write command in order; never --apply, refs fix --yes, or another store write"
---

> [!important] THIS FILE IS AN INSTRUCTION. WHEN REFERENCED IT IS MEANT TO BE TAKEN AS AN ACTION.

Run `flint shard start nsec` if you haven't already.

# Skill: Help With the Migration to v2

Help the operator move a store of the previous version (store format 1, CLI `0.3.0-beta.0` or earlier) and its project files to NUU Secrets v2 (store format 2, CLI `0.4.0-beta.0`). The operator runs each write. You run only the read-only steps, and you give the operator each write command in the order of [[dev-knw-nsec-migrate_v2]].

> [!warning] Operator hold
> On 2026-11-23 the operator said: do not migrate the secrets of the local repositories yet. Use this skill only when the operator asks for the migration now. Else report that the store needs the migration, give the path of the guide, and stop.

# Input

- The store: its name in `secrets store list` and its URL
- Each repository with a `secrets.toml` of this store
- A path for the plan file, outside each repository (for example `~/secrets-migrate-<store>.toml`)
- The instruction of the operator to start the migration now

# Actions

1. **Read the rules and the guide.** Read [[dev-knw-nsec-rules]] (Rules 5, 6, 13, and 15) and [[dev-knw-nsec-migrate_v2]] in full. The rules win over this skill.

2. **Know what you never run.** These are operator writes. Do not run them, also when a hint of the CLI or a `Next:` line names them:
   - `secrets store migrate --apply <plan>`;
   - `secrets refs fix` without `--dry-run` (also `--yes`);
   - `secrets set`, `secrets env <name> <envs>`, `secrets tag …`, `secrets mv`, `secrets expiry set|clear`, and each other store write;
   - `npm install -g …` of the CLI (an agent does not install it);
   - `git tag`, `git push`, `git revert`, and each other Git write in the store or in a backup clone of the store;
   - an edit of `.env.local` or another `.env*` file (Rule 15), and an edit of a plan file;
   - `secrets pull` and `secrets pull --all` during the migration (they write `.env.local`; `secrets pull --check` is a read), and `secrets pull --force`.

3. **Check the backup of the local files first.** The v2 CLI can change `projects.toml` when it loads a project (a v2 `secrets doctor`, or a v2 `--help` in a project folder), and then the old CLI refuses that file. Before each v2 command in a project folder, ask the operator to confirm that step 1 of [[dev-knw-nsec-migrate_v2]] is done on this machine: the backup of `projects.toml` and of each `secrets.local.toml`. The backup is an operator step. Without the confirmation, give the operator the commands of step 1, and stop. Run the version checks below outside each project folder.

4. **Check the versions.**
   - `secrets --version`: it must be `0.4.0-beta.0` or later. A build from source must contain the commit `6ee9194` (the three migration fixes); `1aadb82` is not sufficient. `secrets --help` must list `pull` and `store`, and `secrets store migrate --help` must exist. Else ask the operator to install `npm install -g @nuucognition/secrets-cli@0.4.0-beta.0` (step 2 of the guide).
   - `secrets store list`: the store is in the list.
   - `secrets whoami`: the groups of this machine. The apply needs a machine with `dev`, `prod`, and `deploy`. Report the groups.

5. **Check the store.** Run plain `secrets doctor` in each project folder of the store (Rule 6). Read the `fail` lines:
   - `fail  store format 1 (no store.toml): … needs format 2 …`: the store needs the migration. Continue.
   - `fail  <path>/secrets.toml is not valid:` (or `secrets.local.toml`) with only the lines `env.<VAR>.ref must be secret://<project>/<VARIABLE>, …` and `env.<VAR>.ref: secret://<name> is a secret of the project "<p>": a ref can point only to the project shared or to the project itself …`: these are the old refs of a valid project of the previous version. They are expected until the refs fix (step 8 of the guide). Continue, and list them in the report.
   - `fail  preview: <VAR> → secret://<name> is not in the store` for a ref that is already valid in v2, to a name with a dev value and no preview value: before v2, preview read the dev value. This line is expected before step 7 of the guide. Keep the name for the preview list of step 7. A missing value of another environment is not expected.
   - `ok  store format 2 (store.toml): …`: the store needs no migration (`secrets store migrate --plan` also says `… has the store format 2: it needs no migration.`). Report it and stop. When the project files still have old names, give the operator step 8 of the guide (`secrets refs fix --all … --dry-run` first).
   - Each other `fail` line (another problem of a project file, a key, a recipient, or the store): report it with [[dev-sk-nsec-diagnose]] first. The migration waits. When the operator still has the old CLI, ask the operator to run `secrets doctor` with it in each project: each project must show `0 fail` before the migration ([[dev-knw-nsec-migrate_v2]] § Before you start).

6. **Make the plan (read only).** The plan changes no store file and no project file. It writes only the plan file (names only, never a value). It refreshes the clone: follow the refresh rule of Rule 6, or add `--offline` after a plain `doctor`:

   ```sh
   secrets store migrate --plan --store <store> --projects <repo> --out <plan>
   ```

   Give `--projects` one time for each repository. The path of `<plan>` must have no symbolic link in any part (on macOS, not `/tmp`): else `BAD_PLAN`, and nothing is written. Use a path in the home folder. Do not add `--force` to replace a plan file that the operator edited: use another `--out`, or ask the operator.

7. **Read the plan.** Read the output and the plan file (it holds names only). Report to the operator:
   - the count line (`<n> name(s) of before v2, <n> project(s) in <n> file(s)`);
   - each conflict, by kind, with what the guide says to do (step 5 of the guide);
   - each line with `no preview value` in its comment: after the apply, the operator gives it a preview value or removes `preview` from its set (step 7 of the guide). Propose which one, when the task says if the app has a preview;
   - the names with no rename (`No rename for <n> name(s): …`): the apply writes only their sets and tags;
   - the collection records that become tag records;
   - each `secrets: warning:` line of the plan (a project file that the CLI cannot read).

   Do not edit the plan file. Propose each change (`to`, `envs`, `tags`) with the line, and the operator edits the file.

8. **Give the operator the write commands in order.** Use the exact commands of [[dev-knw-nsec-migrate_v2]], with the real `<store>`, `<url>`, `<repo>`, and `<plan>`:
   1. the backup clone and the tag `before-v2`, and the backup of the local files of each machine (`projects.toml` and each `secrets.local.toml`) (step 1);
   2. the CLI update on each machine and in CI (step 2), and the stop of the writes of the other machines;
   3. the plan edits of step 7 above, if any;
   4. `secrets store migrate --apply <plan>` (step 6 of the guide);
   5. for each name of the preview list: `secrets set <name> --env preview`, or `secrets env <name> <the declared set with no preview>` (step 7). Read the set with `secrets env <name>` (a read: you can run it) and give the exact set: `dev, preview` gives `secrets env <name> dev`; `dev, preview, prod` gives `secrets env <name> dev,prod`. Never add an environment that the set does not have;
   6. `secrets refs fix --all --projects <repo> --plan <plan> --yes`, after the dry run of step 9 below;
   7. `secrets project add <repo>` (step 9);
   8. the old scripts and the old `.env*` lines (steps 10 and 11);
   9. `secrets pull --all` (it writes the `.env.local` files: an operator step here) and the commit of the project files (steps 12 and 14);
   10. the steps of each other machine.

   Give them as one numbered list. Say that the apply is one commit and one push to the team store, and that the rollback is in the guide.

9. **After the apply of the operator: the dry run of the refs.** When the operator says that the apply is done, run:

   ```sh
   secrets refs fix --all --projects <repo> --plan <plan> --dry-run
   ```

   Give the operator the list (`<VAR>  secret://<old> → …`) and the command with `--yes`. Do not run it. A line that the CLI does not change (a warning) goes to the operator.

10. **After the refs fix of the operator: the checks.** Run only these:
   - `secrets project add <repo> --dry-run`: report the projects to add (the operator runs it without `--dry-run`, step 9 of the guide);
   - `secrets pull --check --all`: report each `line <n>: <VAR>` outside the block. The operator removes these lines. Do not open the file;
   - after the operator ran `secrets pull --all`: `secrets pull --check --all` again. Each project must be current;
   - `secrets doctor` in each project folder, with the path-safe check of step 13 of the guide: report the folder, the exit status, and the last line of each, and each `fail` line. The goal is `exit 0` and `0 fail` in each project.

   Do not run `secrets pull --all` in this skill: it writes the `.env.local` files, and each write of the migration is an operator step. A later app run is not a part of this skill.

11. **Find the old prefix.** Search the repositories for the old command, for example `grep -rn --include=package.json '"secret ' <repo>` and a search for `secrets run` in the documents. Report each file and line. You can change a script or a document when the task allows a change of the repository (`"secret next dev"` → `"next dev"`), with the commit rules of the repository. Never add `secrets pull` to the scripts.

12. **Report.** Give the operator: the versions, the groups, the format line, the plan path and its counts, the conflicts, the preview list, the ordered commands, the dry-run list, and the `doctor` results. Never a value.

> [!headless]
> In a headless session, each write of step 8 is a blocker for the operator. Put the ordered commands, the conflicts, and the preview list in the result. Set `flint orbh session set blockers "store <store>: the migration to format 2 needs the operator"`. Do not wait.

# Output

- The versions, the groups of this machine, and the store format line
- The plan file (names only) and the report of its conflicts and its preview list
- The ordered write commands for the operator, from [[dev-knw-nsec-migrate_v2]]
- After the writes of the operator: the dry-run list of the refs, the old `.env*` lines (names and line numbers only), and `secrets doctor` with `0 fail` in each project
