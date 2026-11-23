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
   - `secrets pull --force`.

3. **Check the versions.**
   - `secrets --version`: it must be `0.4.0-beta.0` or later. `secrets --help` must list `pull` and `store`, and `secrets store migrate --help` must exist. Else ask the operator to install `npm install -g @nuucognition/secrets-cli@0.4.0-beta.0` (step 2 of the guide).
   - `secrets store list`: the store is in the list.
   - `secrets whoami`: the groups of this machine. The apply needs a machine with `dev`, `prod`, and `deploy`. Report the groups.

4. **Check the store.** Run plain `secrets doctor` in a project folder of the store (Rule 6). Read the store section:
   - `fail  store format 1 (no store.toml): … needs format 2 …`: the store needs the migration. Continue.
   - `ok  store format 2 (store.toml): …`: the store needs no migration (`secrets store migrate --plan` also says `… has the store format 2: it needs no migration.`). Report it and stop. When the project files still have old names, give the operator step 8 of the guide (`secrets refs fix --all … --dry-run` first).
   - Each other `fail` line: report it with [[dev-sk-nsec-diagnose]] first. The migration waits.

5. **Make the plan (read only).** The plan changes no store file and no project file. It writes only the plan file (names only, never a value). It refreshes the clone: follow the refresh rule of Rule 6, or add `--offline` after a plain `doctor`:

   ```sh
   secrets store migrate --plan --store <store> --projects <repo> --out <plan>
   ```

   Give `--projects` one time for each repository. Do not add `--force` to replace a plan file that the operator edited: use another `--out`, or ask the operator.

6. **Read the plan.** Read the output and the plan file (it holds names only). Report to the operator:
   - the count line (`<n> name(s) of before v2, <n> project(s) in <n> file(s)`);
   - each conflict, by kind, with what the guide says to do (step 5 of the guide);
   - each line with `no preview value` in its comment: after the apply, the operator gives it a preview value or removes `preview` from its set (step 7 of the guide). Propose which one, when the task says if the app has a preview;
   - the collection records that become tag records;
   - each `secrets: warning:` line of the plan (a project file that the CLI cannot read).

   Do not edit the plan file. Propose each change (`to`, `envs`, `tags`) with the line, and the operator edits the file.

7. **Give the operator the write commands in order.** Use the exact commands of [[dev-knw-nsec-migrate_v2]], with the real `<store>`, `<url>`, `<repo>`, and `<plan>`:
   1. the backup clone and the tag `before-v2` (step 1);
   2. the CLI update on each machine and in CI (step 2), and the stop of the writes of the other machines;
   3. the plan edits of step 6 above, if any;
   4. `secrets store migrate --apply <plan>` (step 6 of the guide);
   5. for each name of the preview list: `secrets set <name> --env preview`, or `secrets env <name> dev,prod` (step 7);
   6. `secrets refs fix --all --projects <repo> --plan <plan> --yes`, after the dry run of step 8 below;
   7. `secrets project add <repo>` (step 9);
   8. the old scripts and the old `.env*` lines (steps 10 and 11);
   9. `secrets pull --all` and the commit of the project files (steps 12 and 14);
   10. the steps of each other machine.

   Give them as one numbered list. Say that the apply is one commit and one push to the team store, and that the rollback is in the guide.

8. **After the apply of the operator: the dry run of the refs.** When the operator says that the apply is done, run:

   ```sh
   secrets refs fix --all --projects <repo> --plan <plan> --dry-run
   ```

   Give the operator the list (`<VAR>  secret://<old> → …`) and the command with `--yes`. Do not run it. A line that the CLI does not change (a warning) goes to the operator.

9. **After the refs fix of the operator: the checks.** Run only these:
   - `secrets project add <repo> --dry-run`: report the projects to add (the operator runs it without `--dry-run`, step 9 of the guide);
   - `secrets pull --check --all`: report each `line <n>: <VAR>` outside the block. The operator removes these lines. Do not open the file;
   - `secrets doctor` in each project folder: report the last line of each, and each `fail` line. The goal is `0 fail` in each project.

   After the operator removed the old lines and ran `secrets project add`, you can run `secrets pull --all` and the normal dev command ([[dev-sk-nsec-run_app]]).

10. **Find the old prefix.** Search the repositories for the old command, for example `grep -rn --include=package.json '"secret ' <repo>` and a search for `secrets run` in the documents. Report each file and line. You can change a script or a document when the task allows a change of the repository (`"secret next dev"` → `"next dev"`), with the commit rules of the repository. Never add `secrets pull` to the scripts.

11. **Report.** Give the operator: the versions, the groups, the format line, the plan path and its counts, the conflicts, the preview list, the ordered commands, the dry-run list, and the `doctor` results. Never a value.

> [!headless]
> In a headless session, each write of step 7 is a blocker for the operator. Put the ordered commands, the conflicts, and the preview list in the result. Set `flint orbh session set blockers "store <store>: the migration to format 2 needs the operator"`. Do not wait.

# Output

- The versions, the groups of this machine, and the store format line
- The plan file (names only) and the report of its conflicts and its preview list
- The ordered write commands for the operator, from [[dev-knw-nsec-migrate_v2]]
- After the writes of the operator: the dry-run list of the refs, the old `.env*` lines (names and line numbers only), and `secrets doctor` with `0 fail` in each project
