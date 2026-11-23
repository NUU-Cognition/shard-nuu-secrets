---
description: "The migration guide for a store of the previous version (store format 1, CLI 0.3.0-beta.0 or earlier) to NUU Secrets v2 (store format 2, CLI 0.4.0-beta.0): who must migrate, what changes, the backup, the CLI update on each machine, secrets store migrate --plan and --apply, the preview values, secrets refs fix --all, the project index, the old scripts and .env lines, secrets pull --all, doctor with 0 fail, the other machines, CI, and the rollback. Each step is an operator step"
---

# Knowledge: Migrate a Store of the Previous Version (v2)

This guide is for the operator. It changes a store of the previous version (store format 1) and its project files to NUU Secrets v2 (store format 2). Each write in this guide is an operator write. An agent never runs a write of this guide: it runs only the read-only steps of [[dev-sk-nsec-migrate_v2]] and gives the operator the commands ([[dev-knw-nsec-rules]] § Rule 5 and § Rule 13).

> [!warning] Operator hold
> On 2026-11-23 the operator said: do not migrate the secrets of the local repositories yet. The guide and the skill are for later. Start the migration only when the operator says so.

The words are the words of [[dev-knw-nsec-rules]] § Words. The model of v2 is in [[dev-knw-nsec-model]]. Each command of this guide is in the CLI npm `@nuucognition/secrets-cli@0.4.0-beta.0`, with the same flags ([[dev-knw-nsec-cli]]).

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
- **The plan file.** Keep it outside each repository, for example `~/secrets-migrate-<store>.toml`. It holds names only, never a value.
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

2. **Update the CLI on each machine first.** On each machine of the team, also on each CI machine that runs `secrets plan` or `secrets push`:

   ```sh
   npm install -g @nuucognition/secrets-cli@0.4.0-beta.0
   secrets --version
   ```

   Give the exact version: an npm tag can give an older version. After this step, no machine may write the store with the old CLI.

3. **Check the store.** Run `secrets store list` (the store is in the list) and `secrets doctor --online`. The `fail` line `store format 1 (no store.toml)` is expected. Each other `fail` line must be fixed first ([[dev-sk-nsec-diagnose]]).

4. **Make the plan.** It changes no store and no project file. It reads the names of the store files (no decrypt) and the refs of the projects:

   ```sh
   secrets store migrate --plan --store <store> --projects <repo> --out <plan>
   ```

   - The output: `Store <store> has the store format 1 @ <revision>: <n> name(s) of before v2, <n> project(s) in <n> file(s).`, a table `OLD NAME`, `NEW NAME`, `ENVS`, `TAGS`, `USED BY`, the collection records that become tag records, the conflicts, and the path of the plan file.
   - The rules of the new names: a name of one project goes to `<project>/<VARIABLE>`; a name of two or more projects goes to `shared/<VARIABLE>`; a credential gets its fixed name and the set `deploy`; a Convex deploy key goes to the project whose `[targets.convex] deployment` is its deployment, with the tag `#convex/deploy-keys`. Each name gets the tag of its old collection. A name with a dev value declares `preview` (before v2, preview read the dev value).
   - The plan file exists already: add `--force` to replace it. Without `--force`: `PLAN_EXISTS`.

5. **Read the conflicts, and edit the plan.** The kinds of conflict:

   | Conflict | What to do |
   |---|---|
   | One new name for two or more old names | The plan gives each old name a name from its path (for example `shared/NUU_ACCESS_DEMO_CLERK_JWT_ISSUER_DOMAIN`). When the values are the same, keep one name: change `to` of one line, and give the other lines other names. Or remove the copies after the migration. |
   | Two or more variable names for one secret | The new name comes from the old path (for example `account/url` → `shared/ACCOUNT_URL`). Each variable becomes a link to it. Change `to` when you want another name. |
   | Platform credentials | A Convex deploy key whose deployment has no project, or two or more projects. Set `to` to `<project>/CONVEX_DEPLOY_KEY_<ENV>` of the correct project. |
   | Refs to a name with no value in the store | The apply moves nothing in the store for it. Step 8 changes its refs (with `--plan`). |
   | Names of the store with no ref in a project | Check the new name. A name with no ref goes to `<p>/<REST>` for `apps/<p>/<rest>`, else to `shared/<PATH>`. |

   Each line of `[moves]` in the plan file is `"<old name>" = { to = "<new name>", envs = [...], tags = [...] }`, with a comment line of facts above it (the projects, the variables, the environments with a value, and `no preview value`). Change only `to`, `envs`, and `tags`:

   - `envs`: keep each environment that has a value now. Remove `preview` from a secret that has no preview (then step 7 has no line for it).
   - `tags`: the tags with no `#`, for example `["clerk"]` or `["convex/deploy-keys"]`.
   - When the store changed after the plan, make the plan again (step 4 with `--force`), and check it again.

6. **Apply the plan.** One commit (`store migrate: the store format 2`) and one push:

   ```sh
   secrets store migrate --apply <plan>
   ```

   - The output: one line `moved  <old> → <new>  (<envs>)` for each name, `no value  <old> → <new>  (refs only: …)` for a name with refs and no value, `tag records  #<tag> …  (from collections.toml)`, then `Store <store> has the store format 2 @ <revision>: <n> name(s) moved, <n> declared set(s) written, <n> name(s) tagged.`, then the preview list (step 7), then the `Next:` line.
   - The commit moves each name in each store file (one `move` line of the change log each), writes the declared sets that are not the default (`env-set-change`), the tags (`tag-add`), the tag records from `collections.toml` (`tag-record-change`; `collections.toml` goes), the expiry dates with the names, and `store.toml` with `format = 2`.
   - The refusals (exit 1, nothing written): `NOT_FORMAT_1` (the store has format 2 already), `PLAN_STALE` (the store has a name that the plan does not have: make the plan again), `PLAN_MISMATCH` (the plan does not fit the store; the lines name each problem), `STORE_MISMATCH` (`--store` names another store than the plan), `BAD_TAG`, `NO_PLAN`, and `BAD_PLAN` (the plan file is missing, is a symbolic link, is too large, or cannot be read).

7. **Give each preview secret its own value.** The apply lists each secret that declares `preview` and has no preview value: `<n> secret(s) declare preview and have no preview value (before v2, preview read the dev value):`. For each name of the list, do one of these:

   ```sh
   secrets set <name> --env preview        # the preview value (before v2, it was the dev value)
   secrets env <name> dev,prod              # or: remove preview from its set
   ```

   Type the value at the hidden prompt, or give it on stdin. Never give a value as an argument.

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

10. **Remove the old prefix from the scripts and the documents.** Search each repository for the `secret` prefix and for `secrets run`, for example in the `scripts` of `package.json`, a `Makefile`, a README, or a note. Use the normal command (`"dev": "secret next dev"` → `"dev": "next dev"`). Run `secrets pull` by hand before the dev command: do not put it in `package.json` (`predev`) or in CI. An old command now gives `error: unknown command 'run'`.

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

13. **Check each project.** Each line must show `0 fail`:

    ```sh
    cd <repo> && for d in $(git ls-files '*secrets.toml' | xargs -n1 dirname); do (cd "$d" && secrets doctor | tail -1); done
    ```

    Expected warnings: an `optional` ref with no value in the store (`the local project <p> has <n> ref(s) to its own store project, and the store has no secret of <p>`), and on macOS `full-disk encryption is not checked on darwin`.

14. **Commit the project files.** Commit only the changed `secrets.toml` files and the changed scripts and documents of each repository, with the normal commit process of the repository. Never commit `.env.local`, another `.env*` file with values, `secrets.local.toml`, or the plan file.

## Each other machine of the team

After step 14 of the migration machine:

1. Install the CLI `0.4.0-beta.0` (step 2), if it is not installed yet.
2. `secrets store sync` (it shows the change-log lines of the migration).
3. `git pull` in each repository.
4. `secrets project add <repo>`.
5. `secrets refs fix --all --projects <repo> --dry-run`, then the same with `--yes`. This changes only the `secrets.local.toml` files of this machine (the `secrets.toml` files are changed already). Give `--plan <plan>` too when a local ref has no `move` line (copy the plan file from the migration machine: it holds names only).
6. Remove the old managed lines of the `.env*` files (step 11).
7. `secrets pull --all`, then `secrets doctor` in each project (0 fail).

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
5. The block of `.env.local` stays. The old CLI does not read it, and a program can still read it. Remove it when you want.

## The test of this guide

Task 035 followed this guide on a test store of format 1 made with the CLI `0.3.0-beta.0` (canon `a67936c`), in a scratch folder with test values only, two projects (`web` and `api`), a shared name, a credential (`vercel/token`), a collection record, an expiry date, an optional ref with no value, an old `secret` script, and a managed line in `.env.local`. The CLI of the steps was the build of `Repos/secrets` at `1aadb82`. The result:

- The plan: 4 old names, 2 projects in 2 files, 1 conflict (a ref with no value). The plan edit removed `preview` from `db/url`.
- The apply: 3 names moved, 1 declared set, 3 names tagged, the tag record `#clerk`, `store.toml` with `format = 2`, one commit.
- `refs fix --all --projects <repo> --plan <plan>`: 4 refs in 2 files (2 links to `shared`, 2 implicit refs). A second run: `No ref needs a fix.`
- `secrets pull --all` stopped on the old line of `.env.local` (`line 1: CLERK_SECRET_KEY`) until step 11 removed it. Then both projects pulled.
- `secrets doctor` in each project: `23 ok, 2 warn, 0 fail, 0 skipped` (the optional ref with no value, and full-disk encryption on darwin).
- `secrets plan vercel --env prod --offline`: `key  deploy.enc.yaml → shared/VERCEL_TOKEN  ok`, 0 errors.
- The rollback: the revert of the migration commit and of the project commit; the old CLI ran again, and the new CLI refused the old names.

## Known limits

1. The plan does not read comments, and `refs fix` does not change them.
2. A CLI of `0.3.0-beta.0` or earlier does not know `store.toml`: step 2 is the only protection.
3. The apply needs one machine that opens each store file (`dev`, `prod`, `deploy`).
4. The README of the CLI (`apps/secrets-cli/README.md`, commit `543ebdf` of `even-macbook`) has the sections of the store format and of `secrets store migrate`. This guide gives the order of the steps for a team, and `secrets store migrate --help` gives the options.
5. The review of Task 029 (2026-11-23) found 3 defects in the build `1aadb82`. Two are fixed on `even-macbook`: `refs fix --all --plan <plan>` changes only the refs of the store of the plan (F1, `5afea88`), and the plan file is read and written with no symbolic link in any part of its path (F2, `168725d`; a link gives `BAD_PLAN`). The fix of F3 is in progress: a lowercase variable whose old name is already `<project>/<variable>` gives a plan that the apply refuses. Use a CLI with all three fixes (the release `0.4.0-beta.0`). With an older build: register only the store of the migration on the migration machine (`secrets store list`), write the plan to a folder with no symbolic link in its path, and read the `--dry-run` list of step 8 line by line. The commands and the options of this guide do not change.
