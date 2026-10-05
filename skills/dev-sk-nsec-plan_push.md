---
description: "Prepare a push to a platform (Vercel, Convex, Railway): run secrets plan, act on its statuses and errors, request the credentials, and give the operator the exact plan and push commands; never run push and never add --yes"
---

> [!important] THIS FILE IS AN INSTRUCTION. WHEN REFERENCED IT IS MEANT TO BE TAKEN AS AN ACTION.

Run `flint shard start nsec` if you haven't already.

# Skill: Plan a Push to a Platform

Prepare the delivery of the variables of one environment to a platform. You run `secrets plan`. The operator runs `secrets push`. No value goes through you.

# Input

- The target: `vercel`, `convex`, or `railway`
- The environment: `dev`, `preview`, or `prod`
- The project folder (it has `secrets.toml` with the `[targets.<target>]` table, or a parent folder has it)
- (Optional) Network state: offline or online

# Actions

1. **Read the rules.** You read [[dev-knw-nsec-rules]], also Rule 8 and Rule 9. You never run `secrets push`, for any environment. You never add `--yes`.
2. **Check the mapping.** The mapping needs the `[targets.<target>]` table ([[dev-knw-nsec-model]] § The target tables).
   - When the table is missing, `plan` stops with `✘ <path> has no [targets.<target>] table.` Report it. Do not add the table unless the operator or the task gives its content.
   - When a command stops with `✘ <path> is not valid:` and a `targets.…` problem, use [[dev-sk-nsec-diagnose]]. A table that is not valid stops every command of the project.
3. **Keep the refresh rule.** `plan` without `--offline` refreshes the store clone (fetch and reset). When interrupted local work can matter, first run plain `secrets doctor --env <env>` with the same mapping and store options, and keep its lines about unpushed commits, changed files, and links. Or use `secrets plan … --offline`: it makes no network call and clones nothing. With no clone, `--offline` stops.
4. **Run the plan.** In the project folder, run:
   ```sh
   secrets plan <target> --env <env>
   ```
   - Add `--json` when you need one JSON value. It has the same facts and no values.
   - Never use `--env deploy`: the CLI refuses it.
5. **Read the plan.**
   - `target …`: check the project and the platform environment (Vercel `dev → development`, `preview → preview`, `prod → production`; Railway from the `environments` table).
   - `key …` lines: act on each status.

     | Status | Action |
     |---|---|
     | `ok` | Nothing. |
     | `not set` (optional credential, for example `vercel/team-id`) | Nothing. Without a team id, the requests go to the personal account of the token. |
     | `missing`, `empty` | Request the credential from the operator (step 7). |
     | `not valid` | Give the operator the error line. For example: a Convex key of the wrong type (`prod:` is only for `prod`; `dev:` is for `dev` and `preview`), a Convex key with no type, or a team id that is not `team_…`. Never move a key from one name to another. |
     | `no access` | Expected for a key that is not in group `deploy`. Report it. It is not a permission to add access, and you do not ask for group `deploy` unless the operator or the task says so. |

   - `set <VAR> ← <name>` or `← literal`: the push sends this variable. `remove <VAR> ← <name> (empty, optional)`: the push removes the variable from the platform. On Railway, a removal has the note `Railway can start a deployment for a removal`.
   - `error …` lines:
     - a required `ref` with no value or an empty value: request the value with [[dev-sk-nsec-request_value]] (`secrets set <name> --env <env>`);
     - an empty literal: report it; the operator decides the mapping;
     - `targets.railway.environments has no entry for env <env>: …`: report it; the operator decides the Railway environment (Rule 8);
     - `this machine cannot open deploy.enc.yaml: …`: see `no access` above.
   - The last line is `<n> variables · <m> errors`. With an error, the exit status is 1, and a push would send nothing.
6. **Know what the plan cannot see.** `plan` calls no platform. A successful plan does not prove that the push passes:
   - Vercel: a variable of the same name that covers the environment and also another environment or a custom environment stops the whole push.
   - Railway: a reference variable (`${{…}}`) is `skipped`; the store value does not go to Railway.
   - Any target: a store value or a credential of any length inside an id of the platform stops the push ([[dev-knw-nsec-cli]] § Known limits, item 10).
7. **Request the credentials.** Agents never write a credential. Give the operator the exact command for each missing or empty credential, in the shape of [[dev-sk-nsec-request_value]] (step 3), with the environment `deploy`:

   | Target | Command for the operator | What the operator enters |
   |---|---|---|
   | `vercel` | `secrets set vercel/token --env deploy` | A Vercel access token. |
   | `vercel` | `secrets set vercel/team-id --env deploy` | The team id `team_…` (optional). |
   | `convex` | `secrets set convex/<deployment>/<env> --env deploy` | A typed deploy key of that deployment: `prod:…` for `prod`, `dev:…` for `dev` and `preview`. |
   | `railway` | `secrets set railway/token --env deploy` | A Railway account token or workspace token, not a project token. |

   Only a machine in group `deploy` can open `deploy.enc.yaml`.
8. **Give the operator the commands.** When the plan has no error, write this request. Do not add `--yes`:

   ```text
   Delivery of <project> to <target>, env <env>.
   Plan (secrets plan <target> --env <env>): <n> variables · 0 errors.
     set: <VAR>, <VAR>, …
     remove: <VAR>, … (or: none)
   The plan cannot see the platform state (shared Vercel variables, Railway reference variables, identifier collisions).

   When you approve, run in the project folder:
     secrets plan <target> --env <env>
     secrets push <target> --env <env>
   For --env prod, the push needs --yes. Add it only when you approve this push to production.
   Then send me the push output (names only).
   ```

9. **Read the push output that the operator gives you.**
   - `set`, `removed`, `unchanged`: done. `skipped <VAR> (reference variable)`: Railway keeps its reference; the store value is not on Railway.
   - `failed <VAR> (<reason>)`: the platform refused the write, or the push did not send it. The reason is a fixed text. Do not try to get the message of the platform from another source. Give the operator the line and stderr.
   - `uncertain <VAR> (<reason>; the platform can have the change)`, with `secrets: Check each uncertain variable on the platform before you run the push again.`: never suggest that the push runs again at once. Show the operator the confirmed writes and the uncertain writes from stderr. The operator checks each uncertain variable on the platform first.
   - `… from the platform holds the value of <VAR> (a value that is not secret can be a literal in secrets.toml)` or `… holds the credential <name>`: the push stopped before that request. The operator decides. A value that is not secret can become a literal. Never move a secret value into `secrets.toml`.
   - A shared Vercel variable: the operator changes it in Vercel.
10. **After the push.** Give the operator the `next` and `note` lines. You do not deploy.
    - Vercel: `preview` and `production` read the new values only after a new deployment (`vercel deploy --prod` or `vercel deploy`, run by the operator). `development` needs no deployment. Never run `vercel env pull`: it writes a file with values.
    - Convex: the deployment reads the new values at once.
    - Railway: set variables are saved with no deployment. `Railway can start a deployment for a removal`. The operator makes sure that a deployment of the service runs with the new values.
11. **Headless session.** Put the plan result and the request of step 8 in your result ([[dev-hinit-nsec]] § Result Delivery). A push is never a blocker that you can clear yourself.

# Output

- The plan result: names, statuses, and errors, never a value
- One request to the operator with the missing credentials and the exact `plan` and `push` commands, with no `--yes`
- No push run by you, and no value in your output, a file, or a command argument
