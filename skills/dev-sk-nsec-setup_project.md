---
description: "Prepare a project for NUU Secrets: write secrets.toml with init-project and bind (refs and literals), cover .env* and secrets.local.toml in .gitignore, run doctor, and give the operator the commands for the missing values"
---

> [!important] THIS FILE IS AN INSTRUCTION. WHEN REFERENCED IT IS MEANT TO BE TAKEN AS AN ACTION.

Run `flint shard start nsec` if you haven't already.

# Skill: Set Up a Project

Map the environment variables of one app to secret names and literals in a committed `secrets.toml`. You write names and targets only. The operator writes the values.

# Input

- The app folder (the folder where the app starts)
- The store URL, from the operator, the task, or the `secrets.toml` of another project of the team. For NUU Cognition, the store repository is `NUU-Cognition/nuu-secrets`.
- (Optional) The project name. The default is the name of the app.
- (Optional) The platform targets that the task names (`vercel`, `convex`, `railway`)

# Actions

1. **Read the model and the rules.** You read [[dev-knw-nsec-rules]] and [[dev-knw-nsec-model]]. You write no value in this skill.
2. **Look for a mapping.** Look for `secrets.toml` in the app folder and its parent folders.
   - When it exists, do not run `secrets init-project`. Go to step 4.
   - When it does not exist, go to step 3.
3. **Write the mapping.** In the app folder, run:
   ```sh
   secrets init-project --name <project> --store <url>
   ```
   - The project name starts with a letter or a digit. The other characters are letters, digits, `.`, `_`, or `-`. It has 100 characters or fewer.
   - Never put a credential in the URL. Use an SSH URL (`git@github.com:<org>/<repo>.git`) or an HTTPS URL without user information.
   - If you do not know the store URL, stop and ask the operator. Do not guess.
4. **List the variables of the app.** Get the variable names from the source code, the framework configuration, and the documentation of the app. Do not open a `.env*` file, also not `.env.example` (see [[dev-knw-nsec-rules]] § Rule 2). This command prints only the names that the code reads with `process.env.<VAR>` or `import.meta.env.<VAR>`. It skips `.env*` files:
   ```sh
   grep -rhoE '(process\.env|import\.meta\.env)\.[A-Za-z_][A-Za-z0-9_]*' --exclude='.env*' --exclude-dir=node_modules --exclude-dir=.git . | sed -E 's/.*\.//' | sort -u
   ```
   Add the names that the framework or the documentation of the app gives. When you are not sure about a name, ask the operator for the names only.
5. **Sort each variable.** Use [[dev-knw-nsec-model]] § What goes in the store, and what does not:
   - A secret, or a value that is different in each environment: a `ref` entry.
   - A value that is not secret and is the same in all environments: a literal.
   - A variable that another tool writes (for example `CONVEX_DEPLOYMENT`): no entry.
   - When you cannot tell if a value is secret, use a `ref` entry.
6. **Select the secret names.** A secret name is a lowercase path, usually `<provider>/<what>`, for example `clerk/secret-key` or `sentry/dsn`.
   - The team has one store. Use the same name for the same value in all projects. Read the `secrets.toml` of other projects of the team for names that exist.
   - Never put the environment in the name. The environment selects the store file.
7. **Bind each variable.** Run one command for each variable:
   ```sh
   secrets bind CLERK_SECRET_KEY clerk/secret-key --to local
   secrets bind SENTRY_DSN sentry/dsn --to local --optional
   secrets bind LOG_LEVEL --value info --to local
   ```
   - Use `--optional` only when the app works without the variable. When the store has no value for an optional name, `secret` does not set the variable and also removes a shell variable of that name (unless `--keep-env`).
   - Use `--value` only for a literal that is not secret. Never give a value to `--value`. The secret check of `bind` is not complete ([[dev-knw-nsec-cli]] § Known limits, item 2).
   - Use `--to local` unless the task names a platform target. A platform target needs a `[targets.<target>]` table in `secrets.toml`. Add that table only when the operator or the task gives its content. `bind` refuses a target without its table.
   - stdout says `added`, `replaced`, or `unchanged` for each entry.
   - To remove an entry, delete its line in `[env]` by hand. No command removes an entry. The file holds no values.
8. **Cover the ignored files.** The `.gitignore` of the project (or of the repository) must cover `.env*` and `secrets.local.toml`. Add the missing lines:
   ```gitignore
   .env*
   secrets.local.toml
   ```
   When Git tracks an example file such as `.env.example`, add `!.env.example` after `.env*`.
9. **Run the checks.** In the app folder, run `secrets doctor` with one `--env` for each environment that the task needs, for example `secrets doctor --env dev`. Without `--env`, `doctor` checks `dev`, `preview`, and `prod`. Without `--online`, `doctor` reads the cached clone, and the line `cached revision <revision> (<date>)` says how old it is. When the network is available, use `secrets doctor --online --env <env>` for a current result. Act on the result with [[dev-sk-nsec-diagnose]].
   - The `Mapping` section gives `fail` for each required `ref` that has no value: `<env>: <VAR> → secret://<name> is not in the store`.
   - An `optional` ref with no value is not a `fail`. The `ok` line counts it: `<env>: <n> of <m> refs found (<k> optional not set)`.
   - A `fail` in an environment that the task does not need is not a blocker of the task. Report it separately to the operator.
   - The `Project hygiene` section must say that `.gitignore` covers `.env*` and `secrets.local.toml`.
   - A `warn` such as `.env.local has names that secrets.toml manages: <VAR>, …` is a duplicate. Tell the operator. Do not open or change that file.
10. **Request the values.** For each missing value, give the operator the command with [[dev-sk-nsec-request_value]].
    - `preview` reads the dev value when it has no own value. The operator writes a preview value only when it must be different from dev. `doctor` also gives `fail` for `preview` when the dev value is missing: one dev value fixes both.
    - Request `prod` values only when the task needs production.
11. **Check again.** After the operator writes the values, run `secrets doctor --online --env <env>` for the environments of the task. Plain `doctor` reads the cached clone, which does not have the new values yet. Then run the app with [[dev-sk-nsec-run_app]].
12. **Commit.** Commit `secrets.toml` and `.gitignore` with the normal commit process of the repository. Never commit `secrets.local.toml` or a `.env*` file with values.

# Output

- A committed `secrets.toml` with one entry for each variable of the app (names, targets, and literals only)
- `.gitignore` that covers `.env*` and `secrets.local.toml`
- A list of the missing values, with one `secrets set` command for each, sent to the operator
- After the operator wrote the values: a `secrets doctor --online --env <env>` result with no `fail` for each environment that the task needs
