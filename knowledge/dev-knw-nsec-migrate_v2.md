---
description: "The migration guide for a store of the previous version (store format 1, CLI 0.3.0-beta.0 or earlier) to NUU Secrets v2 (store format 2, CLI 0.4.0-beta.0): who must migrate, what changes, the backup, the CLI update on each machine, secrets store migrate --plan and --apply, the preview values, secrets refs fix --all, the project index, the old scripts and .env lines, secrets pull --all, doctor with 0 fail, the other machines, CI, and the rollback. Each step is an operator step"
---

# Knowledge: Migrate a Store of the Previous Version (v2)

This guide is for the operator. It changes a store of the previous version (store format 1) and its project files to NUU Secrets v2 (store format 2). Each write in this guide is an operator write. An agent never runs a write of this guide: it runs only the read-only steps of [[dev-sk-nsec-migrate_v2]] and gives the operator the commands ([[dev-knw-nsec-rules]] § Rule 5 and § Rule 13).

> [!warning] Operator hold
> On 2026-11-23 the operator said: do not migrate the secrets of the local repositories yet. The guide and the skill are for later. Start the migration only when the operator says so.

The words are the words of [[dev-knw-nsec-rules]] § Words. The model of v2 is in [[dev-knw-nsec-model]]. Each command of this guide is in the CLI npm `@nuucognition/secrets-cli@0.4.0-beta.0`, with the same flags ([[dev-knw-nsec-cli]]). Use that release. A build from source must contain the commit `6ee9194` of `NUU-Cognition/secrets` (and so `5afea88` and `168725d`): a build without these three fixes (for example `1aadb82`) can change the refs of another store, follow a symbolic link in the path of the plan file, and refuse a plan with a name that needs no rename.

## Who must migrate

A team must migrate when its store was made or written with the CLI `0.3.0-beta.0` or earlier (shard `nsec` 0.8.0 or earlier). Such a store has format 1: it has no `store.toml`.

The CLI `0.4.0-beta.0` shows it:

- `secrets doctor` has the line `fail  store format 1 (no store.toml): this CLI (0.4.0-beta.0) needs format 2 and refuses each read and each write of this store`, with the remedy `migrate the store first: secrets store migrate --plan`.
- A command that reads values or writes the store stops (exit 1, `STORE_FORMAT`): `✘ This store has format 1. This CLI (0.4.0-beta.0) needs format 2.` and `Migrate the store first: secrets store migrate --plan`. It changes nothing.
- A project file with old names (`secret://clerk/secret-key`) is not valid for v2: `env.<VAR>.ref must be secret://<project>/<VARIABLE>, …`.

On a store of format 1, these commands still run: `secrets doctor` (also `--online`), `secrets whoami`, `secrets store list`, `secrets store sync`, `secrets store remove`, `secrets store clone`, and `secrets store migrate`.

A new store of `0.4.0-beta.0` (`secrets store init`) has format 2 at once. It needs no migration.

## What changes

| Before v2 (format 1) | v2 (format 2) |
|---|---|
| `secrets run` and the alias `secret` give the values to a program. | They are removed. `secrets pull` writes the values into one block of `.env.local`, then the program runs with its normal command ([[dev-sk-nsec-run_app]]). |
| A secret name is a lowercase path (`clerk/secret-key`). | A secret name is `<project>/<VARIABLE>` (`web/CLERK_SECRET_KEY`). A value of two or more projects is in `shared` (`shared/CLERK_SECRET_KEY`), and each project links to it. |
| Each entry of `secrets.toml` has a `ref`. | An entry with no `ref` is an implicit ref to its own project. A `ref` points only to `shared` or to the own project. |
| `preview` reads `dev`, then its own values. | Each environment reads only its own store file. A secret that declares `preview` needs its own preview value. |
| Each name can have a value in each environment. | Each secret declares its environments (default `dev`, `preview`, `prod`; a platform credential `deploy`). A value in an undeclared environment is not delivered. |
| Collections (the first segment of a name) and `collections.toml`. | Tags (`#clerk`, `#convex/deploy-keys`) in `metadata.toml`, and tag records in `tags.toml`. |
| The credentials `vercel/token`, `vercel/team-id`, `railway/token`, and `convex/<deployment>/<env>`. | `shared/VERCEL_TOKEN`, `shared/VERCEL_TEAM_ID`, `shared/RAILWAY_TOKEN`, and `<project>/CONVEX_DEPLOY_KEY_DEV`, `…_PREVIEW`, `…_PROD`. |
| No `store.toml`. | `store.toml` with `format = 2`. The CLI reads it before each read and each write. |

**The old CLI does not know `store.toml`.** A CLI of `0.3.0-beta.0` or earlier does not read `store.toml`, so the format guard cannot stop it. After the migration, a machine with the old CLI could write old names into the store of format 2. Thus update the CLI on each machine before the apply (step 2). From format 2 on, each later format change stops each older CLI.

## Before you start

- **One migration machine.** It must open each store file: its recipient has the groups `dev`, `prod`, and `deploy` (`secrets whoami`). The apply decrypts each file that holds a name of the plan.
- **The repositories.** Clone or pull each repository that has a `secrets.toml` of this store. `git status` must show no change of a `secrets.toml` file.
- **A clean start with the old CLI.** Before step 2, run `secrets doctor` with the old CLI in each project folder. Each project must show `0 fail`. A `fail` line that is there before the migration is not a migration failure: fix it first ([[dev-sk-nsec-diagnose]]).
- **The plan file.** Keep it outside each repository, for example `~/secrets-migrate-<store>.toml`. It holds names only, never a value. The CLI refuses a symbolic link in any part of the path of the plan file (the file and each folder above it): `BAD_PLAN`, and nothing is written. On macOS, `/tmp` is a link to `/private/tmp`, so a plan file in `/tmp` is refused: use a path in your home folder, or the real path (`/private/tmp/…`).
- **The other machines.** Tell the team: no store write from another machine from step 4 until the end of step 12.

In the commands below, `<store>` is the store name of `secrets store list`, `<url>` is its URL, `<repo>` is the folder of one repository (give `--projects` one time for each repository), and `<plan>` is the path of the plan file.

## The steps on the migration machine

1. **Back up the store.** Make a clone of the store, outside the secrets home, and push a tag:

   ```sh
   git clone <url> store-before-v2
   git -C store-before-v2 tag before-v2
   git -C store-before-v2 push origin before-v2
   ```

   Keep the folder `store-before-v2` until the end of the migration. The rollback uses it ([Rollback](#rollback)).

   **Also back up the local files of each machine** (on the migration machine now, and on each other machine before its first command of the v2 CLI). Do this before each command of the v2 CLI, also before a v2 `secrets doctor` or a v2 `--help` in a project folder: **the v2 CLI can change `projects.toml` when it loads a project** (it writes the new fields of the index), and then the old CLI refuses that file. A backup after a v2 command does not have the old index. The migration changes the `secrets.local.toml` files (step 8), and the v2 CLI writes a new form of `projects.toml` (the project index, with the identity of each project). The rollback restores these copies. The files hold names and paths, never a value. Keep the copy private and outside each repository (run the commands in `bash`):

   ```sh
   B=~/secrets-before-v2
   mkdir -p -m 700 "$B"
   cp -p "<secrets home>/projects.toml" "$B/projects.toml"
   cd <repo> && find . -name secrets.local.toml -not -path '*/node_modules/*' -print0 |
     while IFS= read -r -d '' f; do mkdir -p "$B/<repo name>/$(dirname "$f")" && cp -p "$f" "$B/<repo name>/$f"; done
   ```

   `<secrets home>` is `$SECRETS_HOME`, else `~/.nuucognition/secrets` ([[dev-knw-nsec-model]]). Do the `find` line one time for each repository, with its own `<repo name>`. A machine with no `projects.toml` has no index to back up.

2. **Update the CLI on each machine first.** On each machine of the team, also on each CI machine that runs `secrets plan` or `secrets push`:

   ```sh
   npm install -g @nuucognition/secrets-cli@0.4.0-beta.0
   secrets --version
   ```

   Give the exact version: an npm tag can give an older version. Do not use a build without the commit `6ee9194` (see the start of this guide). Run `secrets --version` outside each project folder. After this step, no machine may write the store with the old CLI.

3. **Check the store.** Run `secrets store list` (the store is in the list), then `secrets doctor --online` in each project folder. Until step 8, the v2 CLI gives these `fail` lines for a valid project of the previous version. They are expected:

   - `store format 1 (no store.toml): this CLI (0.4.0-beta.0) needs format 2 …` (section Store).
   - `<path>/secrets.toml is not valid:` (also `<path>/secrets.local.toml`), with only these lines:
     - `env.<VAR>.ref must be secret://<project>/<VARIABLE>, for example secret://shared/<VARIABLE>` (an old lowercase name);
     - `env.<VAR>.ref: secret://<name> is a secret of the project "<p>": a ref can point only to the project shared or to the project itself (move the value to shared/<x>)` (an old name whose first segment is not the project).
   - `preview: <VAR> → secret://<name> is not in the store` (section Mapping), for a ref that is already valid in v2 (for example `secret://web/token` in the project `web`) to a name with a dev value and no preview value. Before v2, `preview` read the dev value. In v2, `preview` reads only its own values, so this line is expected before step 7. The apply lists the name in the preview list, and step 7 resolves it (a preview value, or the set with no `preview`).

   Each other `fail` line is a blocker: stop, and fix it first ([[dev-sk-nsec-diagnose]]). For example, a missing value of `dev` or `prod`, or a missing preview value of a name with no dev value, is not an expected line. The project must be `0 fail` with the old CLI (see [Before you start](#before-you-start)). Step 7 gives the preview values, step 8 changes the refs, and step 13 needs `0 fail` in each project.

4. **Make the plan.** It changes no store and no project file. It reads the names of the store files (no decrypt) and the refs of the projects:

   ```sh
   secrets store migrate --plan --store <store> --projects <repo> --out <plan>
   ```

   - The output: `Store <store> has the store format 1 @ <revision>: <n> name(s) of before v2, <n> project(s) in <n> file(s).`, a table `OLD NAME`, `NEW NAME`, `ENVS`, `TAGS`, `USED BY`, the names with no rename (`No rename for <n> name(s): <names> (each one is already a v2 name; the apply writes only its set and its tags).`), the collection records that become tag records, the conflicts, and the path of the plan file.
   - The rules of the new names: a name of one project goes to `<project>/<VARIABLE>`; a name of two or more projects goes to `shared/<VARIABLE>`; a credential gets its fixed name and the set `deploy`; a Convex deploy key goes to the project whose `[targets.convex] deployment` is its deployment, with the tag `#convex/deploy-keys`. Each name gets the tag of its old collection. A name with a dev value declares `preview` (before v2, preview read the dev value). An old name that is already a v2 name (for example `web/token` for the variable `token` of the project `web`) needs no rename: its line has `to` = its own name.
   - The plan file exists already: add `--force` to replace it. Without `--force`: `PLAN_EXISTS`.
   - A symbolic link in the path of `--out` (for example `/tmp` on macOS): `BAD_PLAN`, and nothing is written. Give a real path.

5. **Read the conflicts, and edit the plan.** The kinds of conflict:

   | Conflict | What to do |
   |---|---|
   | One new name for two or more old names | The plan gives each old name a name from its path (for example `shared/NUU_ACCESS_DEMO_CLERK_JWT_ISSUER_DOMAIN`). When the values are the same, keep one name: change `to` of one line, and give the other lines other names. Or remove the copies after the migration. |
   | Two or more variable names for one secret | The new name comes from the old path (for example `account/url` → `shared/ACCOUNT_URL`). Each variable becomes a link to it. Change `to` when you want another name. |
   | Platform credentials | A Convex deploy key whose deployment has no project, or two or more projects. Set `to` to `<project>/CONVEX_DEPLOY_KEY_<ENV>` of the correct project. |
   | Refs to a name with no value in the store | The apply moves nothing in the store for it. Step 8 changes its refs (with `--plan`). |
   | Names of the store with no ref in a project | Check the new name. A name with no ref goes to `<p>/<REST>` for `apps/<p>/<rest>`, else to `shared/<PATH>`. |

   Each line of `[moves]` in the plan file is `"<old name>" = { to = "<new name>", envs = [...], tags = [...] }`, with a comment line of facts above it (the projects, the variables, the environments with a value, and `no preview value`). Change only `to`, `envs`, and `tags`:

   - `envs`: keep each environment that has a value now. Remove `preview` from a secret that has no preview, and keep the other environments of the line (`["dev", "preview"]` → `["dev"]`; `["dev", "preview", "prod"]` → `["dev", "prod"]`). Then step 7 has no line for it.
   - A line whose `to` is its own name needs no rename. The apply writes only its `envs` and its `tags`. You can give it a new name.
   - `tags`: the tags with no `#`, for example `["clerk"]` or `["convex/deploy-keys"]`.
   - When the store changed after the plan, make the plan again (step 4 with `--force`), and check it again.

6. **Apply the plan.** One commit (`store migrate: the store format 2`) and one push:

   ```sh
   secrets store migrate --apply <plan>
   ```

   - The output: one line `moved  <old> → <new>  (<envs>)` for each moved name, `kept  <name>  (no rename; <envs>)` for each name with no rename, `no value  <old> → <new>  (refs only: …)` for a name with refs and no value, `tag records  #<tag> …  (from collections.toml)`, then `Store <store> has the store format 2 @ <revision>: <n> name(s) moved, <n> name(s) with no rename, <n> declared set(s) written, <n> name(s) tagged.` (the part `with no rename` only when there is such a name), then the preview list (step 7), then the `Next:` line. With `--json`, the names with no rename are in the field `stayed`.
   - The commit moves each moved name in each store file (one `move` line of the change log each; a name with no rename gets no `move` line, and the commit message has the line `<name> (no rename)`), writes the declared sets that are not the default (`env-set-change`), the tags (`tag-add`), the tag records from `collections.toml` (`tag-record-change`; `collections.toml` goes), the expiry dates with the names, and `store.toml` with `format = 2`.
   - The refusals (exit 1, nothing written): `NOT_FORMAT_1` (the store has format 2 already), `PLAN_STALE` (the store has a name that the plan does not have: make the plan again), `PLAN_MISMATCH` (the plan does not fit the store; the lines name each problem), `STORE_MISMATCH` (`--store` names another store than the plan), `BAD_TAG`, `NO_PLAN`, and `BAD_PLAN` (the plan file is not a regular file, is too large, or cannot be read; a symbolic link in any part of its path; a folder that does not exist or that changes during the command).

7. **Give each preview secret its own value.** The apply lists each secret that declares `preview` and has no preview value: `<n> secret(s) declare preview and have no preview value (before v2, preview read the dev value):`. For each name of the list, do one of these:

   - Give it its own preview value: `secrets set <name> --env preview`. Type the value at the hidden prompt, or give it on stdin. Never give a value as an argument.
   - Or remove `preview` from its set, and keep each other environment of the set. Read the set first:

     ```sh
     secrets env <name>                       # <name>  declared <set> [(the default)]  values <envs>  store …
     ```

     Then give the set with no `preview`:

     | The declared set now | The command |
     |---|---|
     | `dev, preview, prod` | `secrets env <name> dev,prod` |
     | `dev, preview` | `secrets env <name> dev` |
     | `preview, prod` | `secrets env <name> prod` |

     Do not add an environment: a set with `prod` and no prod value gives a `fail` line for `prod` in `doctor`.

8. **Change the refs of each repository.** First the dry run, then the change:

   ```sh
   secrets refs fix --all --projects <repo> --plan <plan> --dry-run
   secrets refs fix --all --projects <repo> --plan <plan> --yes
   ```

   - The dry run shows `<n> ref(s) in <m> file(s) moved:`, then each file, and one line `<VAR>  secret://<old> → secret://<new>` (a link) or `<VAR>  secret://<old> → implicit (<project>/<VARIABLE>)` (an implicit ref: the `ref` field goes) for each ref, then `Dry run: no file was changed.`
   - `--plan` gives the new name to a ref whose old name has no `move` line (a ref with no value in the store).
   - The change keeps each other byte of the file. It does not change comments: a comment that names an old name stays. Change it by hand, or later.
   - With no terminal and no `--yes`, it changes nothing: `✘ secrets refs fix changes <n> project file(s) and needs a confirmation. No file was changed.` (`CONFIRMATION_NEEDED`).
   - A second run gives `No ref needs a fix.`

9. **Add the projects to the project index.** The index skipped the project files of the old grammar:

   ```sh
   secrets project add <repo> --dry-run
   secrets project add <repo>
   ```

   Without this step, `doctor` warns for each store project: `the store project <p> has no local project on this machine`.

10. **Remove the old prefix from the scripts and the documents.** Search each repository for the `secret` prefix and for `secrets run`, for example in the `scripts` of `package.json`, a `Makefile`, a README, or a note. Use the normal command (`"dev": "secret next dev"` → `"dev": "next dev"`). Run `secrets pull` by hand before the dev command: do not put it in `package.json` (`predev`) or in CI. An old `secrets run` command now gives `Unknown command: run` (exit 1), and the old `secret` command is not found.

11. **Remove the old managed lines of the `.env*` files.** Before v2, some people copied values into `.env.local` or `.env`. `secrets pull` stops on such a line (`PULL_MANAGED_OUTSIDE`, with `line <n>: <VAR>`). Find them without a write:

    ```sh
    secrets pull --check --all
    ```

    Remove each line that it names (the operator edits the file; an agent never opens it). `secrets doctor` names the managed names of the other `.env*` files.

12. **Pull.** In each repository:

    ```sh
    secrets pull --all
    ```

    Each project gets the line `secrets: pull <project> · env dev · store <revision> (<date>) · <n> variables → <path>/.env.local`.

13. **Check each project.** Run the check in `bash`. It is safe for a path with spaces, and it gives the exit status of `doctor` for each project:

    ```sh
    cd <repo> && git ls-files -z -- '*secrets.toml' | while IFS= read -r -d '' f; do
      d=$(dirname "$f"); out=$(cd "$d" && secrets doctor 2>&1); s=$?
      printf '%s  exit %s  %s\n' "$d" "$s" "$(printf '%s\n' "$out" | tail -n 1)"
    done
    ```

    Each line must show `exit 0` and `0 fail`. A line with `exit 1` names the project: run `secrets doctor` in that folder, and read the `fail` lines.

    Expected warnings: an `optional` ref with no value in the store (`the local project <p> has <n> ref(s) to its own store project, and the store has no secret of <p>`), and on macOS `full-disk encryption is not checked on darwin`.

14. **Commit the project files.** Commit only the changed `secrets.toml` files and the changed scripts and documents of each repository, with the normal commit process of the repository. Never commit `.env.local`, another `.env*` file with values, `secrets.local.toml`, or the plan file.

## Each other machine of the team

After step 14 of the migration machine:

1. Back up the local files of this machine (step 1: `projects.toml` and each `secrets.local.toml`) before the first command of the v2 CLI. Then install the CLI `0.4.0-beta.0` (step 2), if it is not installed yet.
2. `secrets store sync` (it shows the change-log lines of the migration).
3. `git pull` in each repository.
4. `secrets project add <repo>`.
5. `secrets refs fix --all --projects <repo> --dry-run`, then the same with `--yes`. This changes only the `secrets.local.toml` files of this machine (the `secrets.toml` files are changed already). Give `--plan <plan>` too when a local ref has no `move` line (copy the plan file from the migration machine: it holds names only).
6. Remove the old managed lines of the `.env*` files (step 11).
7. `secrets pull --all`, then the check of step 13 (each project `exit 0` and `0 fail`).

A machine that still has the old CLI must not write the store.

## CI

- CI does not pull: it has no key for `dev` or `preview` values on the disk.
- The file `deploy.enc.yaml` stays. The apply moves the credentials to their new names in it (`shared/VERCEL_TOKEN`, `shared/VERCEL_TEAM_ID`, `shared/RAILWAY_TOKEN`, `<project>/CONVEX_DEPLOY_KEY_<ENV>`), with the set `deploy`.
- `secrets plan` and `secrets push` work again after step 8 and the commit of step 14, with the CLI `0.4.0-beta.0` on the CI machine. Before the first push, run `secrets plan <target> --env <env>` and check each `key` line (`ok`).
- Pin the exact CLI version in CI: `npm install -g @nuucognition/secrets-cli@0.4.0-beta.0`.

## Rollback

Use the rollback when the migration has a problem and the team must go back to the previous version. The rollback is simple only before the first store write after the migration.

1. **Stop the writes** of each machine.
2. **Revert the migration commit** in the backup clone of step 1 (the operator):

   ```sh
   git -C store-before-v2 pull
   git -C store-before-v2 log --oneline --grep '^store migrate: the store format 2'
   git -C store-before-v2 revert --no-edit <the commit of the migration>
   git -C store-before-v2 push origin main
   ```

   The revert removes `store.toml` and `tags.toml`, and brings back the old names and `collections.toml`. The store has format 1 again. A store write after the migration (for example a preview value of step 7) is a commit after the migration commit: revert it first, newest first. The tag `before-v2` shows the old state: compare with `git -C store-before-v2 diff before-v2 HEAD --stat`.
3. **Revert the project commits** of step 14 in each repository (`git revert`), so that the refs have the old names again.
4. **Install the old CLI** on each machine: `npm install -g @nuucognition/secrets-cli@0.3.0-beta.0`. Then `secrets store sync`. The CLI `0.4.0-beta.0` refuses the store of format 1 again.
5. **Restore the local files** of each machine from the backup of step 1. The revert of step 3 does not change them: `secrets.local.toml` is not in Git, and the old CLI refuses the v2 refs of a `secrets.local.toml` (`env.<VAR>.ref must be secret://<name>, where <name> is a lowercase path like clerk/secret-key`) and the new fields of `projects.toml` (`an unknown field (use path, name, store, identity, seen)`):

   ```sh
   B=~/secrets-before-v2
   cp -p "$B/projects.toml" "<secrets home>/projects.toml"
   cd "$B/<repo name>" && find . -name secrets.local.toml -print0 |
     while IFS= read -r -d '' f; do cp -p "$f" "<repo>/$f"; done
   ```

   The copy of `projects.toml` keeps the identity of each project. A `secrets.local.toml` that is not in the backup was made after the migration: remove it, or change its refs to the old names. Then run `secrets doctor` with the old CLI in each project folder: each project must show `0 fail`.
6. The block of `.env.local` stays. The old CLI does not read it, and a program can still read it. Remove it when you want.

## The test of this guide

Task 035 followed this guide two times on a test store of format 1 made with the CLI `0.3.0-beta.0` (the code of canon `a67936c`), in a scratch folder with test keys and test values only. The second run (the fix round of the review, 2026-11-23) used the CLI source of `Repos/secrets` at `010304c` (the fixes `5afea88`, `168725d`, and `6ee9194`). The fixture: three projects (`web`, `api`, and `space-app` in the folder `apps/space app`, a path with a space), a name of three projects, a credential (`vercel/token`), a collection record, an expiry date, an optional ref with no value, a name with no rename (`web/token`), a dev-only name (`db/url`), an old `secret` script, a `secrets.local.toml` with an old ref, and a managed line in `.env.local`. The result of the second run:

- Before step 2, the old CLI gave `0 fail` in each project. Step 1 backed up the store (the clone and the tag) and the local files (`projects.toml` and the `secrets.local.toml` of `web`, in a folder with mode 700).
- Step 3: the v2 CLI gave 2 `fail` lines in each project, only the expected ones (the store format 1, and the old refs of `secrets.toml` and `secrets.local.toml`). The loop of step 13 gave `exit 1` for each of the three projects, also for `apps/space app`.
- The plan: `No rename for 1 name(s): web/token`. A plan path with a symbolic link was refused, and no file was written.
- The apply: `4 name(s) moved, 1 name(s) with no rename, 2 declared set(s) written, 5 name(s) tagged`, the line `kept  web/token  (no rename; dev)`, one commit, and the preview list of 4 names.
- Step 7: a preview value for `shared/CLERK_SECRET_KEY`; `secrets env <name>` read the sets, then `secrets env space-app/SPACE_KEY dev,prod`, `secrets env web/DB_URL dev`, and `secrets env web/token dev` (no environment was added).
- `refs fix --all --projects <repo> --plan <plan>`: a second dry run gave `No ref needs a fix.`
- `secrets pull --check --all` named `line 1: CLERK_SECRET_KEY` of the old `.env.local`. After step 11 removed it, `secrets pull --all` wrote the three projects.
- Step 13 (bash): `apps/api  exit 0  doctor: 24 ok, 1 warn, 0 fail, 0 skipped`, `apps/space app  exit 0  doctor: 24 ok, 1 warn, 0 fail, 0 skipped`, and `apps/web  exit 0  doctor: 25 ok, 1 warn, 0 fail, 0 skipped`.
- `secrets plan vercel --env prod --offline` passed. No test value was in the CLI output, the plan, or a Git object; only the `.env.local` files (mode 600) had values.
- The rollback: the revert of the later store writes and of the migration commit (no difference to the tag `before-v2`), the revert of the project commit, `secrets store sync` with the old CLI, then the restore of `projects.toml` and `secrets.local.toml` from the backup. The old CLI then gave `0 fail` in each project, and the v2 CLI refused the store of format 1 again.

The third run (the second fix round, 2026-11-23) added the project `stable` with the ref `secret://stable/token`, which is already valid in v2, and with dev and prod values and no preview value. The CLI source was at `11f984e` (the migration code of `6ee9194` and `010304c`). The result:

- `secrets --version` outside a project folder did not change `projects.toml`. The first v2 `doctor` in a project folder changed it, and the old `secrets project list` then exited 1. Thus step 1 backs up the local files before each v2 command.
- Step 3: `stable` gave only the store format line and `fail  preview: token → secret://stable/token is not in the store` (the expected preview line). The plan had `No rename for 2 name(s): stable/token, web/token`. Step 7 gave `secrets env stable/token dev,prod`.
- Step 13: each of the four projects gave `exit 0` and `0 fail`.
- After the rollback and the restore, the old CLI gave `0 fail` in each project, and the old `secrets project list` exited 0.

## Known limits

1. The plan does not read comments, and `refs fix` does not change them.
2. A CLI of `0.3.0-beta.0` or earlier does not know `store.toml`: step 2 is the only protection.
3. The apply needs one machine that opens each store file (`dev`, `prod`, `deploy`).
4. The README of the CLI (`README.md` of `NUU-Cognition/secrets`, commit `010304c` of `even-macbook`) has the sections of the store format and of `secrets store migrate`. This guide gives the order of the steps for a team, and `secrets store migrate --help` gives the options.
5. The review of Task 029 (2026-11-23) found 3 defects in the build `1aadb82`. All three are fixed on `even-macbook`: `refs fix --all --plan <plan>` changes only the refs of the store of the plan (`5afea88`); the plan file is read and written with no symbolic link in any part of its path (`168725d`); a name that needs no rename is not a move, and each plan of `--plan` parses (`6ee9194`). Use the release `0.4.0-beta.0`, which has these fixes, or a build that contains `6ee9194`. Do not use `1aadb82`.
6. The plan does not back up the local files of a machine (`secrets.local.toml`, `projects.toml`). Step 1 does it.
