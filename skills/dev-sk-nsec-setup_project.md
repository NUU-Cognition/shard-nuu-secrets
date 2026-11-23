---
description: "Prepare a project for NUU Secrets with the CLI only: write secrets.toml with init-project (it also adds the .gitignore lines), find the variables with vars, bind implicit refs, links to shared, and literals (a value that another project has moves to shared: the operator decides and runs the move), set the targets that the operator gives, run doctor, give the operator the commands for the missing values, then pull the values into the block of .env.local"
---

> [!important] THIS FILE IS AN INSTRUCTION. WHEN REFERENCED IT IS MEANT TO BE TAKEN AS AN ACTION.

Run `flint shard start nsec` if you haven't already.

# Skill: Set Up a Project

Map the environment variables of one app to secret names and literals in a committed `secrets.toml`. You write names and targets only, and only through the commands of the CLI. The operator writes the values.

When this machine has no key or no store yet, the setup of the machine comes first: see [[dev-knw-nsec-setup]]. `secrets setup --print` shows which steps remain.

# Input

- The app folder (the folder where the app starts)
- The store URL, from the operator, the task, or the `secrets.toml` of another project of the team. For NUU Cognition, the store repository is `NUU-Cognition/nuu-secrets`.
- (Optional) The project name. The default is the name of the app.
- (Optional) The platform targets that the task names (`vercel`, `convex`, `railway`), with their ids

# Actions

1. **Read the model and the rules.** You read [[dev-knw-nsec-rules]] and [[dev-knw-nsec-model]]. You write no value in this skill. You do not edit `secrets.toml`, `secrets.local.toml`, or the NUU Secrets lines of `.gitignore` by hand (Rule 4).
2. **Look for a mapping.** Look for `secrets.toml` in the app folder and its parent folders.
   - When it exists, do not run `secrets init-project`. Run `secrets project gitignore` in the app folder (it adds only the missing lines). Go to step 4.
   - When it does not exist, go to step 3.
3. **Write the mapping.** In the app folder, run:
   ```sh
   secrets init-project --name <project> --store <url>
   ```
   - It also adds the missing lines `.env*` and `secrets.local.toml` to the `.gitignore` of the app folder. stdout has two lines: `created  <path>  …` and `added  .env*, secrets.local.toml  to <path>/.gitignore` (or `unchanged  …`). Do not use `--no-gitignore` unless the operator asks.
   - The project name starts with a letter or a digit. The other characters are letters, digits, `.`, `_`, or `-`. It has 100 characters or fewer.
   - Never put a credential in the URL. Use an SSH URL (`git@github.com:<org>/<repo>.git`) or an HTTPS URL without user information. The CLI refuses a URL that is not a usual form ([[dev-knw-nsec-setup]] § Store URL forms).
   - If you do not know the store URL, stop and ask the operator. Do not guess.
4. **List the variables of the app.** In the app folder, run:
   ```sh
   secrets vars --unmapped
   ```
   - It prints the names that the source code reads and no mapping has, with the first file of each. It never reads a `.env*` file and never shows a source line. Do not open a `.env*` file, also not `.env.example` (see [[dev-knw-nsec-rules]] § Rule 2).
   - `secrets vars` (with no option) also shows the names in state `mapped`, `local`, and `tool`. A `tool` name gets no entry.
   - Add the names that the framework or the documentation of the app gives. When you are not sure about a name, ask the operator for the names only.
5. **Sort each variable.** Use [[dev-knw-nsec-model]] § What goes in the store, and what does not:
   - A secret, or a value that is different in each environment: a store entry (step 6).
   - A value that is not secret and is the same in all environments: a literal.
   - A variable that another tool writes (for example `CONVEX_DEPLOYMENT` and `NEXT_PUBLIC_CONVEX_URL`, which `npx convex dev` writes to `.env.local`): no entry with the target `local`. The tool owns its line, outside the block of `secrets pull`: a managed name on that line stops each pull.
   - A password, a login, or recovery codes that no program reads from its environment: no entry. It belongs in the vault, and only the operator writes it ([[dev-knw-nsec-rules]] § Rule 14).
   - When you cannot tell if a value is secret, use a store entry.
6. **Select the secret names.** A secret name is `<project>/<VARIABLE>`. The project is the `[project] name` of the `secrets.toml`, or `shared`.
   - **A new variable uses an implicit ref.** The entry has no `ref`: it names `<project>/<VARIABLE>` of its own project, for example `nuu-track/CLERK_SECRET_KEY` for `CLERK_SECRET_KEY` in the project `nuu-track`. `secrets bind <VAR> --to local` writes it (step 8).
   - **A value that is already in `shared`** gets a link: `secrets bind <VAR> shared/<NAME> --to local`. Run `secrets list --project shared` (no key needed) to see the shared secrets. The variable name of a link can be different from `<NAME>`.
   - **A value that another project already has moves to `shared`.** Run `secrets list` (no key needed): when another project has the same value (for example the same provider key in `other-app/OPENAI_API_KEY`), do not write a second copy, and do not link to the other project: a ref can point only to `shared` or to the own project (`bind` refuses it with `BAD_LINK`). Tell the operator the name and the projects. The operator decides and runs the move (`secrets mv <project>/<VARIABLE> shared/<VARIABLE>`, then `secrets refs fix` in the other project). This is an operator write ([[dev-knw-nsec-rules]] § Rule 13). After the move, bind the link. Until then, the variable stays unbound, and you report it.
   - **Do not migrate.** Do not change the names of other projects, do not move their secrets, and do not change the tags or the environment sets. A store of the previous version (lowercase path names, for example `clerk/secret-key`) needs the migration: the guide is [[dev-knw-nsec-migrate_v2]]. The migration is an operator write.
   - **Declared environments.** A new secret declares `dev`, `preview`, `prod` by default. When the app needs a value only in some environments, read the current set first (`secrets env <name>`: `declared <set>`), and tell the operator the new set: the current set minus the environments that the app does not need. Never add an environment that the current set does not have. The operator changes it.
   - Never put the environment in the name. The environment selects the store file.
7. **Set the platform targets (only when the task names them).** A platform target needs its table before an entry can use it. Use the ids that the operator or the task gives (Rule 8). Never guess an id.
   ```sh
   secrets target set vercel --project <name-or-id>
   secrets target set convex --deployment <short-name>
   secrets target set railway --project <name-or-id> --service <name-or-id> --env prod=<railway-env>
   ```
   Then run `secrets doctor`: a table that is not valid stops every command of the project, also `secrets pull`.
8. **Bind each variable.** Run one command for each variable:
   ```sh
   secrets bind CLERK_SECRET_KEY --to local                       # the implicit ref <project>/CLERK_SECRET_KEY
   secrets bind SENTRY_DSN shared/SENTRY_DSN --to local --optional   # a link to a value in shared
   secrets bind LOG_LEVEL --value info --to local
   ```
   - Use `--optional` only when the app works without the variable. When the store has no value for an optional name, `secrets pull` does not write the variable into the block.
   - Use `--value` only for a literal that is not secret. Never give a value to `--value`. The secret check of `bind` is not complete ([[dev-knw-nsec-cli]] § Known limits, item 2).
   - Use `--to local` unless the task names a platform target (step 7). `bind` refuses a target without its table.
   - Do not give the target `convex` to a name that the Convex tool owns (for example `CONVEX_DEPLOYMENT` or `NEXT_PUBLIC_CONVEX_URL`): the mapping is then not valid.
   - An entry for this machine only goes to `secrets.local.toml`: `secrets bind <VAR> [shared/<NAME>] --local`. Use it only when the operator asks. `bind --local` refuses when Git does not ignore `secrets.local.toml`.
   - To remove an entry: `secrets unbind <VAR>` (or `--local`). Never remove an entry to make a run pass.
   - stdout says `added`, `replaced`, or `unchanged` for each entry.
   - The delivery to a platform is in [[dev-sk-nsec-plan_push]]. Only the operator runs `secrets push`.
9. **Check the ignored files.** `secrets doctor` (step 10) says in `Project hygiene` whether `.gitignore` covers `.env*` and `secrets.local.toml`. When it does not, run `secrets project gitignore`. `secrets pull` refuses a `.env.local` that Git does not ignore (`PULL_NOT_IGNORED`) or that Git tracks (`PULL_TRACKED`). For a tracked `.env.local`, tell the operator: `git rm --cached .env.local` needs the approval of the operator, and the Git history keeps the old file. An ignore line does not change a file that Git tracks already: when `doctor` says that Git tracks `secrets.local.toml`, run `git rm --cached secrets.local.toml`, and tell the operator that the Git history keeps the old file.
10. **Run the checks.** In the app folder, run `secrets doctor` with one `--env` for each environment that the task needs, for example `secrets doctor --env dev`. Without `--env`, `doctor` checks `dev`, `preview`, and `prod`. Without `--online`, `doctor` reads the cached clone, and the line `cached revision <revision> (<date>)` gives the cached commit and its commit date. It does not give the last refresh time. After this plain check, when the network is available, run the same check with `--online` for a current result. Act on the result with [[dev-sk-nsec-diagnose]].
    - The `Mapping` section gives `fail` for each required store entry that has no value in a declared environment: `<env>: <VAR> → secret://<name> is not in the store`.
    - An `optional` ref with no value is not a `fail`. The `ok` line counts it: `<env>: <n> of <m> refs found (<k> optional not set)`.
    - A `fail` in an environment that the task does not need is not a blocker of the task. Report it separately to the operator.
    - A `warn` such as `.env.local has names that secrets.toml gives to local outside the block of NUU Secrets: <VAR>, … (line <n>)` is a managed name outside the block, and each pull stops on it. Tell the operator each name and line number. Do not open or change that file, and do not delete the line without the approval of the operator ([[dev-knw-nsec-rules]] § Rule 15). The same applies to `<file> has names that secrets.toml manages: <VAR>, …` for another `.env*` file.
11. **Request the values.** For each missing value, give the operator the command with [[dev-sk-nsec-request_value]].
    - `preview` does not read `dev`. A secret that declares `preview` needs its own preview value: request it as its own value (`secrets set <name> --env preview`), also when it is the same as the dev value. When the app has no preview, read the current set (`secrets env <name>`), and tell the operator the set with no `preview`: remove only `preview`, and keep each other environment. `dev, preview, prod` gives `secrets env <name> dev,prod`; `dev, preview` gives `secrets env <name> dev`. Never add `prod`: a declared `prod` with no prod value gives a `fail` line in `doctor`, and a removal in a prod plan. The operator changes the set.
    - Request `prod` values only when the task needs production.
    - The operator can also run `secrets setup` in the app folder: its values step asks for each missing dev value with a hidden prompt.
12. **Check again.** After the operator writes the values, run plain `secrets doctor` for the environments of the task. Keep the lines about local changes, unpushed commits, and links. Then add `--online` and run the same check. A successful write on this machine already updates its clone. A write on another machine can require a refresh. Then run `secrets pull --check` and `secrets pull`, and run the app with its normal command ([[dev-sk-nsec-run_app]]).
13. **Commit.** Commit `secrets.toml` and `.gitignore` with the normal commit process of the repository. Never commit `secrets.local.toml`, `.env.local`, or another `.env*` file with values. Do not add `secrets pull` to the scripts of `package.json`: CI has no key.

# Output

- A committed `secrets.toml` with one entry for each variable of the app (names, targets, and literals only), written with the commands of the CLI
- `.gitignore` that covers `.env*` and `secrets.local.toml`
- A list of the missing values, with one `secrets set` command for each, sent to the operator
- After the operator wrote the values: a plain `secrets doctor --env <env>` check, then the same check with `--online`, with no `fail` for each environment that the task needs, and a current block (`secrets pull --check` says `current`)
