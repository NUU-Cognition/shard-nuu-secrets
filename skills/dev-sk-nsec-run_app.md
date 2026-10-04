---
description: "Run a program with the values of the store in its process environment (secret <cmd>), read the secret: line and the exit status, and act on a failure"
---

> [!important] THIS FILE IS AN INSTRUCTION. WHEN REFERENCED IT IS MEANT TO BE TAKEN AS AN ACTION.

Run `flint shard start nsec` if you haven't already.

# Skill: Run an App with Secrets

Run a program (a dev server, a test, a script) with the variables of its project from the store. The values go only into the process environment of the program. The wrapper writes no file with values. It can clone or refresh the encrypted store clone, and it can update Git and index files.

# Input

- The program and its arguments, for example `pnpm dev` or `node server.js`
- The project folder (it has `secrets.toml`, or a parent folder has it)
- (Optional) The environment. The default is `dev`.
- (Optional) Network state: offline or online

# Actions

1. **Check the rules.** You read [[dev-knw-nsec-rules]]. The program must not print its process environment or a value. Do not start it with debug or dump options.
2. **Go to the app folder.** Run the command in the folder of the app, or in a child folder of the `secrets.toml`. When you cannot, add `--mapping <path>` or `--project <name>` (not both).
3. **Select the environment.**
   - Use the default `dev` for development and tests.
   - Use `--env preview` only when the task needs preview values.
   - Do not use `--env prod` or `--env deploy` unless the operator asks for it. These are production values and platform credentials.
4. **Build the command.** Put the options of `secret` before the program:
   ```sh
   secret pnpm dev
   secret --env preview node server.js
   secret --offline pnpm test
   secrets run --env preview -- node server.js
   ```
   - Use `--offline` when there is no network. With an existing clone, `--offline` does not refresh it, and stderr names its cached revision. When no clone exists, this build still clones the store: that is a network access. Before an offline run, run `secrets store list` and find the row whose `URL` is the `store` of the mapping. When there is no such row and the task forbids network access, do not run: report a blocker.
   - Use `--keep-env` only when a shell variable must win over the store value. Normally the store value wins, and stderr names the replaced variable.
   - `--` ends the options of `secret`. The first word that is not an option also ends them. Each word after that goes to the program without a change.
5. **Do not put `secret` in project files.** Do not add `secret` to the scripts of `package.json`, to a `Makefile`, or to CI files. The person or the agent adds `secret` in front of the command at run time. A project file with `secret` fails on a machine without NUU Secrets.
6. **Turborepo.** When the program is `turbo run …` and Turborepo uses strict env mode, Turborepo can drop the variables. Run `secret` in the app folder with the app command (for example `secret next dev`). If the task needs `turbo run`, ask the operator to add the variables to `passThroughEnv` or `globalPassThroughEnv` in `turbo.json`. Do not change `turbo.json` yourself unless the task says so.
7. **Long-running programs.** A dev server does not stop. Run it in the background of your harness. Stop it when the task is done. `secret` gives SIGTERM to the program and waits for it.
8. **Read the `secret:` line.** Before the program starts, stderr has one line:
   `secret: <project> · env <env> · store <revision> (<date>) · <n> variables`
   - Check the project, the environment, and the count.
   - `· <n> optional not set` means that the store has no value for `<n>` `optional` entries. The CLI does not give these variables. But a variable of that name that is already in your shell environment stays in the process environment of the program, also without `--keep-env`. The `secret:` line does not prove that the variable is absent.
   - When the task needs an optional variable to be absent, remove it from the launch environment. Do not show its value: `env -u <VAR> secret <cmd>`.
   - `store not read (no ref entries)` means that the mapping has only literals for this target.
   - `secrets: cloning the store of <project> from <url>` before the line means a first clone. See [[dev-knw-nsec-cli]] § Known limits, item 3.
9. **Check a variable without its value** when the task needs proof:
   ```sh
   secret sh -c 'test -n "$CLERK_SECRET_KEY" && echo "CLERK_SECRET_KEY is set"'
   ```
10. **Read the exit status.**

    | Status | Meaning | Action |
    |---|---|---|
    | `0` | The program succeeded. | Continue. |
    | `1` with `✘ <n> variable(s) have no value in env <env>` | Values are missing. Nothing ran. | Use [[dev-sk-nsec-request_value]] for each listed name. |
    | `1` with `✘ There is no secrets.toml …` | No mapping here. | Go to the project folder, or use [[dev-sk-nsec-setup_project]]. |
    | `1` with `✘ sops decrypt failed …` | This machine cannot open the store file. | Use [[dev-sk-nsec-diagnose]]. Do not run `sops`. |
    | `2` with `✘ No command to run.` | No program after the options. The program did not start. | Give the program. |
    | `1` with `error: …` | The option parser refused an option of `secret` (for example `error: unknown option '--bad'`). | Fix the option, or put it after the program name. |
    | `126` with `✘ Cannot start <cmd> (EACCES).`, `127` with `✘ The program "<cmd>" is not on PATH.` | The program is not executable, or not on `PATH`. | Fix the program name or the path. |
    | `128 + n` | Signal `n` stopped the program (143 is SIGTERM). | Normal when you stopped it. |
    | other | The exit status of the program. | Read the output of the program. |

    After the program starts, `secret` exits with the exit status of the program. That status can also be 1, 2, 3, 126, or 127. Thus read the `✘` or `error:` message, not only the status. The `secret:` line on stderr means that the CLI resolved the variables and then tried to start the program.

11. **If a value shows** in the output of the program, follow [[dev-knw-nsec-rules]] § If a value leaks.

# Output

- The program ran with the variables of its project, or a clear failure with its next step
- No value in your output, in a file, or in a command argument
